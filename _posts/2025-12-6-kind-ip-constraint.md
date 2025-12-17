---
layout: post
title: "What happens when Kind doesn't have enough IP(s)?"
tags: [personal]
comments: false
---

I wanted to write a quick blog to document a tiny experiment I ran last week.  
Just dumping my rough notes as it is.  

What I want to test is a scenario of creating a (Kind) cluster when it doesn't have enough IP addresses, to assign internally etc.  
Meaning I try to create a Kind cluster and give it only a docker bridge network with 20 or 50 IP addresses (basically a very tiny pool of IP(s)).  

Actually, now that I think more, 20-50 IP(s) are actually too much for my experiment.  
Because the docker bridge IP pool will only be used for assigning IP(s) to the kind nodes (the control-plane and worker nodes).  
Inside these kind nodes - for the Pods and for the Containers IP(s), the node will configure its own pool of private IP addresses, so that doesn't come from the docker bridge network from the host.

Therefore, the flow is roughly like:
- (my host network) sets aside a little set of private IP for the docker bridge -> 
- (then docker bridge network) assigns an IP to a node -> 
- (node internal network) which assigns IP(s) to pods and containers

With this understanding now, I feel I should create a docker bridge network with even less IP(s), lets say 5 IP(s).  
And with this newly created docker bridge network, if I try to create a Kind cluster with 5 or more nodes, atleast 1 node will never get an IP.  
And that is exactly the behavior I want to test.

I also know out of these 5 IP(s) -
- one will be used as gateway, so that is gone,
- and then the rest 4 will be assigned to 4 nodes

(Note: this above understanding is incomplete right now, will be fixed later in the post.)

So, with that mathematics done now, let's run the experiment.

---

First off, create a Kind cluster:

```bash
❯ cat kind-config.yaml 
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  - role: worker


❯ kind create cluster --name ip-test --retain --config kind-config.yaml
Creating cluster "ip-test" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-ip-test"
You can now use your cluster with:

kubectl cluster-info --context kind-ip-test

Have a nice day! 👋
```


---


**Number one**

To create a new docker bridge network, I need to run the following command:

```
❯ docker network create --driver bridge  \
    --subnet 172.20.0.0/29  \
    --gateway 172.20.0.1 \
    --aux-address "reserved1=172.20.0.6" \
    kind-small-net
```

Notice the flag I passed, `--subnet 172.20.0.0/29`.  
This translates to a private IP network `172.20.0.0` with a subnet mask of `255.255.255.248/29`.  
This subnet mask will give me exactly 8 IP addresses.  
(and that's the closest I can get to making an IP pool with exact 5 usable IP(s)).

So, how these 8 IP(s) will be used, is explained below:

- `172.20.0.0` = network (not usable)
- `172.20.0.1` → commonly used as gateway (Docker sets a gateway)
- `172.20.0.2 – 172.20.0..6` = usable host addresses (that’s 5 addresses here) 
- `172.20.0.7` = broadcast (not usable)

Also notice, that I restricted one of the available 5 IP addresses using the flag, `--aux-address "reserved1=172.20.0.6"` to create an IP contrained scenario.


---


**Number two**

Kind always create a default docker network bridge with name "kind" automatically.  

So, I can try to create a new docker network bridge (`kind-small-net`) with constrained IP pool like we did above.  
And then try to connect existing Kind cluster nodes to this newly created bridge network.

Like following:

```bash
❯ for c in $(docker ps --filter "name=ip-test" -q); do   docker network connect kind-small-net $c; done
Error response from daemon: no available IPv4 addresses on this network's address pools: kind-small-net (15f712efffba69730048b9f826e3e68702646a37df4d09c86288fca47a5a52f6)
```

Notice, I got an error - `no available IPv4 addresses on this network's address pools`.  
So far, everything is going as expected.

Now, let's check if anything happened to the cluster:

```
❯ kubectl get nodes -o wide
NAME                    STATUS     ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
ip-test-control-plane   NotReady   control-plane   9m4s    v1.34.0   172.18.0.6    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker          NotReady   <none>          8m48s   v1.34.0   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker2         NotReady   <none>          8m49s   v1.34.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker3         NotReady   <none>          8m49s   v1.34.0   172.18.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker4         NotReady   <none>          8m49s   v1.34.0   172.18.0.5    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3


❯ docker container ps -a
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                       NAMES
aca02654b39a   kindest/node:v1.34.0   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes                               ip-test-worker
a7da65a5a3f7   kindest/node:v1.34.0   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes                               ip-test-worker4
4605b3acc669   kindest/node:v1.34.0   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes                               ip-test-worker3
856fbf8aec0d   kindest/node:v1.34.0   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes   127.0.0.1:46031->6443/tcp   ip-test-control-plane
fcb796be89c8   kindest/node:v1.34.0   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes                               ip-test-worker2
```

NO! All nodes still has an IP assigned to them.  
And none of these assigned IP(s) are from our newly created network bridge (atleast not in the `kubectl` output).    
(And yes, I also see the `NotReady` status, so that is something.)

What's going on? Let's inspect both the network bridges:

```bash
❯ docker network inspect kind

        "Name": "kind",
         ...
        "Scope": "local",
        "Driver": "bridge",
         ...
        "IPAM": {
            ...
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                },
            ]
        },
        ...
        "Containers": {
            "4605b3acc6699173bd975a3d6b74d25e688eecfce644962bd7fb26c50d42f890": {
                "Name": "ip-test-worker3",
                ...
                "IPv4Address": "172.18.0.3/16",
            },
            "856fbf8aec0d287fea50ce9255260a22a1f707e12ea34aac313e3b356ffc3d8d": {
                "Name": "ip-test-control-plane",
                ...
                "IPv4Address": "172.18.0.6/16",
            },
            "a7da65a5a3f7dc78947e33d8f797854c47630adbee68ea30a46590a5238862ac": {
                "Name": "ip-test-worker4",
                ...
                "IPv4Address": "172.18.0.5/16",
            },
            "aca02654b39a931e9d27e313f61a78719eb389cb008e86f078c184fac2bae4e7": {
                "Name": "ip-test-worker",
                ...
                "IPv4Address": "172.18.0.2/16",
            },
            "fcb796be89c800e6e6107026ab04f448b9efa889957c10253f181bd63fa88075": {
                "Name": "ip-test-worker2",
                ...
                "IPv4Address": "172.18.0.4/16",
            }
        },
```

and

```bash
❯ docker network inspect kind-small-net

        "Name": "kind-small-net",
         ...
        "Scope": "local",
        "Driver": "bridge",
         ...
        "IPAM": {
            ...
            "Config": [
                {
                    "Subnet": "172.20.0.0/29",
                    "Gateway": "172.20.0.1",
                    "AuxiliaryAddresses": {
                        "reserved1": "172.20.0.6"
                    }
                }
            ]
        },
        ...
        "Containers": {
            "4605b3acc6699173bd975a3d6b74d25e688eecfce644962bd7fb26c50d42f890": {
                "Name": "ip-test-worker3",
                ...
                "IPv4Address": "172.20.0.4/29",
            },
            "856fbf8aec0d287fea50ce9255260a22a1f707e12ea34aac313e3b356ffc3d8d": {
                "Name": "ip-test-control-plane",
                ...
                "IPv4Address": "172.20.0.5/29",
            },
            "a7da65a5a3f7dc78947e33d8f797854c47630adbee68ea30a46590a5238862ac": {
                "Name": "ip-test-worker4",
                ...
                "IPv4Address": "172.20.0.3/29",
            },
            "aca02654b39a931e9d27e313f61a78719eb389cb008e86f078c184fac2bae4e7": {
                "Name": "ip-test-worker",
                ...
                "IPv4Address": "172.20.0.2/29",
            }
        },
```

OK, so, 4 out of the 5 nodes (1 control-plane + 3 workers) are assigned an IP from the new `kind-small-net` network.

But the entire set (1 control-plane + 4 workers) are still assigned an IP from the default `Kind` network.

Let's try one more thing.  
Just like the `docker network connect` command, there's also a command to disconnect containers from a network as well.  
Let's run that as well and see if that makes the Kind cluster nodes fall back to the new `kind-small-net` network IP(s).

```bash
❯ for c in $(docker ps --filter "name=ip-test" -q); do   docker network disconnect kind $c; done

❯ docker network inspect kind

        "Name": "kind",
        ...
        "Scope": "local",
        "Driver": "bridge",
        ...
        "IPAM": {
            ...
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                },
                ...
            ]
        },
        ...
        "Containers": {},
        ...


❯ kubectl get nodes -o wide
NAME                    STATUS     ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
ip-test-control-plane   NotReady   control-plane   28m   v1.34.0   172.18.0.6    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker          NotReady   <none>          28m   v1.34.0   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker2         NotReady   <none>          28m   v1.34.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker3         NotReady   <none>          28m   v1.34.0   172.18.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker4         NotReady   <none>          28m   v1.34.0   172.18.0.5    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3

❯ for c in $(docker ps --filter "name=ip-test" -q); do   docker network disconnect kind $c; done
Error response from daemon: container aca02654b39a931e9d27e313f61a78719eb389cb008e86f078c184fac2bae4e7 is not connected to network kind
Error response from daemon: container a7da65a5a3f7dc78947e33d8f797854c47630adbee68ea30a46590a5238862ac is not connected to network kind
Error response from daemon: container 4605b3acc6699173bd975a3d6b74d25e688eecfce644962bd7fb26c50d42f890 is not connected to network kind
Error response from daemon: container 856fbf8aec0d287fea50ce9255260a22a1f707e12ea34aac313e3b356ffc3d8d is not connected to network kind
Error response from daemon: container fcb796be89c800e6e6107026ab04f448b9efa889957c10253f181bd63fa88075 is not connected to network kind
```

Ok! So, all Kind nodes are indeed disconnected from the default "Kind" network now.  
But still, they have an IP assigned from the old "Kind" network only (i.e, it didn't fall back to the newly created bridge "kind-small-net").

So, looks like Kind only look for the default automatically created docker bridge network ("Kind") for its cluster configuration.  

And therefore, regardless of me creating a new docker network bridge and attaching existing Kind node containers to that, Kind will always assign IP(s) from this default network bridge to the kind nodes.  
And so, no IP exhaustion scenaio will happen.

---

**Number three**

If I still want the Kind cluster to use a custom IP pool, the way to do that is:
- delete the existing docker network bridge, with name "Kind" (if one is existing), and
- recreate a new one manually, with the same name "Kind", with the required custom tiny IP pool I need.

Like following:

```
❯ docker network rm kind
kind

❯ docker network inspect kind
[]
Error response from daemon: network kind not found


❯ docker network create --driver bridge  \
    --subnet 172.20.0.0/29  \
    --gateway 172.20.0.1 \
    --aux-address "reserved1=172.20.0.6" \
    kind

f64a9d47cf585e9e61c3d25da2b3d3684f02b633b53ee7c053a60c2da0eafd84

❯ docker network inspect kind

        "Name": "kind",
        "Id": "f64a9d47cf585e9e61c3d25da2b3d3684f02b633b53ee7c053a60c2da0eafd84",
        "Created": "2025-12-16T18:35:37.818562813+05:30",
        "Scope": "local",
        "Driver": "bridge",
        ...
        "IPAM": {
            ...
            "Config": [
                {
                    "Subnet": "172.20.0.0/29",
                    "Gateway": "172.20.0.1",
                    "AuxiliaryAddresses": {
                        "reserved1": "172.20.0.6"
                    }
                }
            ]
        },
        ...
        "ConfigOnly": false,
        "Containers": {},
        ...
```

And with the required IP constrained docker bridge network with name "kind" in place, let's create the Kind cluster as following:

```
❯ kind create cluster --name ip-test --retain --config kind-config.yaml
Creating cluster "ip-test" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✗ Preparing nodes 📦 📦 📦 📦 📦  
ERROR: failed to create cluster: command "docker run --name ip-test-worker2 --hostname ip-test-worker2 --label io.x-k8s.kind.role=worker --privileged --security-opt seccomp=unconfined --security-opt apparmor=unconfined --tmpfs /tmp --tmpfs /run --volume /var --volume /lib/modules:/lib/modules:ro -e KIND_EXPERIMENTAL_CONTAINERD_SNAPSHOTTER --detach --tty --label io.x-k8s.kind.cluster=ip-test --net kind --restart=on-failure:1 --init=false --cgroupns=private --volume /dev/mapper:/dev/mapper kindest/node:v1.34.0@sha256:7416a61b42b1662ca6ca89f02028ac133a309a2a30ba309614e8ec94d976dc5a" failed with error: exit status 125
Command Output: a14eea4647334a84e95142893a735d1cf97bcb74def2793de4a6653c6f187cc9
docker: Error response from daemon: failed to set up container networking: no available IPv4 addresses on this network's address pools: kind (f64a9d47cf585e9e61c3d25da2b3d3684f02b633b53ee7c053a60c2da0eafd84)

Run 'docker run --help' for more information
```

OK, we managed to get the scenario working.  
This time, the cluster failed at the bootstrap time only, with the expected error:  

```bash
failed to set up container networking: no available IPv4 addresses on this network's address pools: kind
```

And once again, docker network inspect also shows that node `ip-test-worker2` was the one which failed to get an IP from the pool.  
Plus, the cluster is not responding.

```bash
❯ docker network inspect kind

        "Name": "kind",
        "Id": "f64a9d47cf585e9e61c3d25da2b3d3684f02b633b53ee7c053a60c2da0eafd84",
        "Created": "2025-12-16T18:35:37.818562813+05:30",
        "Scope": "local",
        "Driver": "bridge",
        ...
        "IPAM": {
            ...
            "Config": [
                {
                    "Subnet": "172.20.0.0/29",
                    "Gateway": "172.20.0.1",
                    "AuxiliaryAddresses": {
                        "reserved1": "172.20.0.6"
                    }
                }
            ]
        },
        ...
        "Containers": {
            "2230663632be9887858ac1037b1f01ec856122bf5ab02e6acf9188c6bfb12b32": {
                "Name": "ip-test-worker3",
                ...
                "IPv4Address": "172.20.0.3/29",
            },
            "6be8bb30074055a2049ccb50d066f0b1cd9cf62243f3d3c619b16c0e555dcf80": {
                "Name": "ip-test-control-plane",
                ...
                "IPv4Address": "172.20.0.4/29",
            },
            "7c545d00e21bbea07ed9a22226c80753acf09d3ed574914e711a8ddc67847013": {
                "Name": "ip-test-worker4",
                ...
                "IPv4Address": "172.20.0.2/29",
            },
            "f7b35fb4319944ba1ef7fb426e1708bac1bad0601b29d6cd1f43a2a4acb41233": {
                "Name": "ip-test-worker",
                ...
                "IPv4Address": "172.20.0.5/29",
            }

❯ kubectl get nodes
E1216 18:40:11.021156  144524 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

---

**Number four**

When I ran the "kind create cluster" command above, I learnt there's a flag called `--retain` that will retain the nodes (the respective docker containers) even if cluster bootstrap fails, for debugging purposes.


```bash
❯ docker container ps -a
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
a14eea464733   kindest/node:v1.34.0   "/usr/local/bin/entr…"   6 minutes ago   Created                                    ip-test-worker2
2230663632be   kindest/node:v1.34.0   "/usr/local/bin/entr…"   6 minutes ago   Up 6 minutes                               ip-test-worker3
6be8bb300740   kindest/node:v1.34.0   "/usr/local/bin/entr…"   6 minutes ago   Up 6 minutes   127.0.0.1:38589->6443/tcp   ip-test-control-plane
7c545d00e21b   kindest/node:v1.34.0   "/usr/local/bin/entr…"   6 minutes ago   Up 6 minutes                               ip-test-worker4
f7b35fb43199   kindest/node:v1.34.0   "/usr/local/bin/entr…"   6 minutes ago   Up 6 minutes                               ip-test-worker

❯ docker exec -it ip-test-control-plane /bin/bash
root@ip-test-control-plane:/# exit

❯ for c in $(docker ps -a --filter "name=ip-test" -q); do   docker inspect --format '{{.Name}} {{.NetworkSettings.Networks.kind.IPAddress}}' "$c"; done
/ip-test-worker2 
/ip-test-worker3 172.20.0.3
/ip-test-control-plane 172.20.0.4
/ip-test-worker4 172.20.0.2
/ip-test-worker 172.20.0.5
```

We can see that all containers have an IP assigned to them from our new custom "kind" network bridge, but not the `ip-test-worker2`.

---


**Number five**

Ok, let's finish it with fixing our cluster.

Let's recreate the "kind" network bridge.  
It is going to be a custom network even now, but let's remove the restriction for that last available and usable IP address.

```bash
❯ docker network create --driver bridge      --subnet 172.20.0.0/29      --gateway 172.20.0.1    kind
0272f4e1a9d33eaf31a77a5aec2dece1cf95098345fb1b0bbbcb825901af0c2b

❯ kind create cluster --name ip-test --retain --config kind-config.yaml
Creating cluster "ip-test" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-ip-test"
You can now use your cluster with:

kubectl cluster-info --context kind-ip-test

Have a nice day! 👋

❯ kubectl get nodes -o wide
NAME                    STATUS     ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
ip-test-control-plane   NotReady   control-plane   41s   v1.34.0   172.20.0.6    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker          NotReady   <none>          26s   v1.34.0   172.20.0.5    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker2         NotReady   <none>          26s   v1.34.0   172.20.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker3         NotReady   <none>          26s   v1.34.0   172.20.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
ip-test-worker4         NotReady   <none>          26s   v1.34.0   172.20.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.17.9-1-default   containerd://2.1.3
```

Done! We have the nodes created with IP addresses assigned from the new custom "kind" network pool.

I know the state of the nodes are `NotReady`, but that's not part of this experiment.  
(updated later - I know the reason why all nodes stayed in `NotReady` state. Because I only used a kind-config that disabled default CNI setup 🤦‍♀️. Anyway, removing this should fix it.)

```yaml
networking:
  disableDefaultCNI: true
```

Next, I want to try is to see what happens when I constrain "PodCIDR" and "ServiceCIDR" pools. o/
