---
layout: post
title: "Networking, and Kubernetes Networking!"
tags: [personal]
comments: false
---

This week was a Hack Week (2025 edition) at work (thanks to my employer, SUSE).

The theme, "Remain Curious" - basically, I can do anything as long as I learn something new and do something outside of my regular work responsibilities. :)

I decided to spend the week learning about basic "(Linux) Networking and then Kubernetes Networking".

Most things in Kubernetes, as long as they just work, I haven't paid much attention to how they work (well, now a days I do. So, I want to learn more).  
Kubernetes, as a system, is huge and complex at this point, so understanding every single piece requires its own dedicated time. Networking is same!

My goal was simple! (very simple!)

On my local machine, start a simple (docker) container, and make it accessible via internet.  
So, I started!

First on my machine itself, on the localhost.  
Then on my machine, but on the LAN network.  
Then on my mobile phone, on the same LAN network.  
Then on my mobile phone, on a different network.  
Then on my devices, on my machine's external IP.  
Then on my friend's device sitting in a different city, on my machine's external IP.  
And then via a domain name, which routes to my external IP, on all these devices (local machines, external machines etc).  

And then repeat the same, but now the container runs inside a Kubernetes cluster.

And I decided to use "Kind" to create my Kubernetes cluster.  
So, the container essentially runs inside a container on my host machine.






