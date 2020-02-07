<!--
.. title: Mysterious nginx upstream errors
.. slug: mysterious-nginx-upstream-errors
.. date: 2020-02-07 22:06:59 UTC+01:00
.. tags: web,nginx,docker
.. category: 
.. link: 
.. description: 
.. type: text
-->

For easier web app deploy we use the following stack

1. Ubuntu server
2. [Docker](https://www.docker.com/products/container-runtime)
3. Containerized [Nginx proxy](https://github.com/jwilder/nginx-proxy) with [Letsencrypt companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)
4. Containerized Web app

Docker simplifies the web app development by enabling us, the developers, to pin the majority of the software dependencies versions, and, further, to have the exact same dependencies during development as for the deploy. This greatly reduces the "*it works on my machine*" moments. Those moments do not go away fully, because we usually do not specify the version of every last dependency used, however, they become really rare. One thing we do not test in development, though, is the web app operation when coupled with Nginx. And that sometimes leads to unexpected problems, such as the random upstream errors I will describe in next few lines.
<!-- TEASER_END -->

# The problem

When several containers were running on our machine, using the setup as described above, a strange behavior appeared. Our monitoring tool ([UptimeRobot](https://uptimerobot.com/), use it, it is great and has a free tier), reported that one of the web app containers keeps returning service unavailable error every few minutes. It was not always the same app container, after some time and usually after a machine restart, another app container was chosen to be problematic and there was just no deterministic way to figure out which will be hit next. The Nginx kept reporting an error that looked something like follows:

```log
[error] 27212#0: *314 no live upstreams while connecting to   upstream, client: ip_address , server: example.com, request: "GET / HTTP/1.1", upstream: "http://example.com", host: "example.com", referrer: "http://example.com/mypages/"
```

We used our Google-fu to no success. Apparently no-one else on Github issues or Stackoverflow was having those problems. We just crossed our fingers that the next container "hit" by this strange behavior won't be the one that matters and at the time when it matters, before we address the issue. And that's exactly what happened. The boss wanted an application demonstration, which we could not deliver, because the browser kept showing the 502 - Service unavailable error.

# Getting serious

After that unfortunate demonstration I jumped at the problem with a renewed zeal. There's got to be something we can do. Changing to another proxy seemed a tempting idea and I had my eye on [Traefik](https://containo.us/traefik/) for a while. Traefik seemed like a proxy built for the container age. But that would be our last resort, since it meant changing a bunch of things, and who can guarantee that it will work as promised. Also at the time, Traefik 2 was announced, which would mean that we will have to update everything all over again.

So I tried to systematically analyze our problem in order to finally find some pattern about the frequency of the error or about which container would be hit next. Since Nginx complained that there was no live upstream I `curl`ed the endpoints within the app containers to count how many times the web app from within the container would die. 

# Lucky stumble

Now guess what. None of the containerized web apps ever died and what is more, even UptimeRobot stopped reporting the 502 errors for the time of the test. So I had an idea, I will simply set-up a cron job to periodically `curl` the endpoints of every container running. The following line was born after some trial and error

```bash
docker ps -q | xargs -I{} docker inspect {} | jq '.[0].NetworkSettings.Networks | .[].IPAddress' | xargs -I{} curl -I {}
```

This line, together with `cron` now keeps the lights on and applications running without 502 for several months now and unless someone tells me how to solve the problem properly it will stay that way for a while. 

It nags me that I could not find a proper solution, however:

1. The services are not that critical
2. I am not an expert in the field
3. We ran out of time. 

We are now deep in new projects with new challenges and this one waits until some smarter guy or a gal comes around and fixes it.

