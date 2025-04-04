---
layout: post
title:  "Docker distribution/registry - Or: why version pinning is important"
author: Ivan
date:   2025-04-04 16:22:23 +0300
categories: Docker Registry Distribution OpenTelemetry Keycloak
---

## *Re: Urgent - I get this error*
Thursday afternoon... last thing I'd like to see is a mail with similar subject from the CTO of the company. 
``` shell
$ docker login -u $CONTAINER_REGISTRY_USERNAME -p $CONTAINER_REGISTRY_PASSWORD $CONTAINER_REGISTRY
Error response from daemon: login attempt to https://registry.XXXXXX/v2/ failed with status: 401 Unauthorized
```
For various reasons we run our own private container registry. Needless to say, without it all development is blocked. Worse, still if a running pod needs to be started on another node, the new node MAY not contain the image in it's local cache so a pod migration(a normal event in kubernetes) can actually result in a service outage. I know it's a single point of failure, have a ticket in the backlog about it.

The setup is quite straightforward, Registry is deployed in k8s behind an LB, uses s3 for storage and Keycloak for AA. It's been worry-free for the past year or so, apart from some semi-periodic clean-up of old cruft. 

Looking at the logs I saw a lot of errors: 
```
time="2025-04-04T09:32:55.822077511Z" level=error msg="traces export: Post \"https://localhost:4318/v1/traces\": dial tcp [::1]:4318: connect: connection refused" environment=development go.version=go1.23.7 instance.id=1df5abf0-2cc7-4268-9d28-b3f4a046ce87 service=registry version=3.0.0
```

So why the hell is registry all of a sudden trying to reach *localhost:4318*? I know, by accident, that such a request is actually the service trying to reach out to an [Otel(OpenTelemetry)](https://opentelemetry.io) collector but why would that cause an outage? 

Well, it didn't. Hidden among the thousand similar **errors** there was also another line:
```
time="2025-04-04T09:30:53.436663295Z" level=info msg="failed to verify token: token signed by untrusted key with ID: \"FML4:7GSU:IBQ4:XXX\""
```

Let me sink that for a moment. I got 100+ **errors** that Otel is unable  to post it's traces, but I get a single **info** message why my AA setup is not working!

Google led to this issue: [INFO[0101] failed to verify token: token signed by untrusted key with ID](https://github.com/distribution/distribution/issues/4299). It seems the logic inside Registry has changed and now the Keycloak setup that's been working for some time and that is done exactly as per Keycloack documentation needs additional configuration.

But why did it break now? Well, in the deployment I had:
```yaml
kind: Deployment
metadata:
  name: registry
  ...
spec:
  template:
    spec:
      containers:
      - name: registry
        image: registry:latest
```

And guess what? *:latest* just got bumped to *3.0.0* yesterday. So it was just a matter of time until the pod was restarted for disaster to strike.
For now, I've just changed to **image: registry:2** and that seems to have fixed the issue on prod.

However, I feel kind of alienated by the registry guys because:
- They knowingly broke token auth for *legacy* implementations. No migration path, not even a mention besides *"docker/libtrust has been replaced with go-jose/go-jose in #4096"*. I for sure don't know what *go-jose* is and don't really care as long as it does not make me do more work. When a [fix/pull request was proposed] (https://github.com/distribution/distribution/pull/4417) it was dismissed as pretty much *We used to do bad things but now we do only good ones*.
- OTel traces being enabled by default is a PITA. If it's an opt-in - fine, I'll just not use it. If there's a way to disable it via config file option - also somewhat OK. Luckily it's in k8s so I can just do:
  ```
  configMapGenerator:
  - name: registry-new-config
    literals:
    - REGISTRY_LOG_LEVEL="info"
    - OTEL_TRACES_EXPORTER="none"
  ```
  But what if it runs on real hardware/shared VM? 

Both these issues shows to me that the maintainers of registry just don't care about their users... 