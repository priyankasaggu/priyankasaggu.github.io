---
layout: post
title: "Understanding kubeadm Bootstrap Tokens (through Node Bootstrapping)"
tags: [personal]
comments: false
---

_(This blog are my notes as I read and try to understand the thesis paper, [Usable Access Control in Cloud Management Systems](https://github.com/luxas/research/blob/main/msc_thesis.pdf), written by Lucas Käldström.)_


For a good while now, I have been wanting to understand how a node joins a Kubernetes cluster using the `kubeadm join` command.  
And especially, the part about the symmetric token (that we generate or we get from the control plane node after a successful `kubeadm init` run, and then we share it with all other nodes wanting to join the cluster).

So, things I want to understand are: 
- why there's a symmetric token?
- and what is the role of this token?
- and can I try to create a kubeadm token manually and use that to join an existing Kubernetes cluster successfully?

And as I read more and more of the thesis paper (linked at the top), I am understanding that the Access control mechanism(s) (for authentication, and authorization, and admission) used within the Kubernetes clusters are extremly elaborate and thought out from security perspective (and ofcourse, they're not simple at all, not right away at least).  
So, seeing a simple token passed through the command line in plain text format felt a bit off the place (or too simple right away).

And ofcourse, I have a feeling, that it is not, so, following is me trying to figure out some answers for my questions.

---

I created a simple Kind cluster, and looked for anything "bootstrap" related on the `kube-apiserver-*` pod.

```bash
❯ kind create cluster

❯ kubectl get pod kube-apiserver-kind-control-plane -n kube-system -o yaml | grep "bootstrap"

   - --enable-bootstrap-token-auth=true    
```

I got a flag in the output, `--enable-bootstrap-token-auth=true`.  
And I don't know what this flag actually does.

So, below I'm trying to create a simple docker container with the `kube-apiserver` image and look at the `kube-apiserver --help` menu for the definition of the flag.

```bash
❯ kubectl get pod kube-apiserver-kind-control-plane -n kube-system -o yaml | grep "image:"
   image: registry.k8s.io/kube-apiserver:v1.34.0
   image: registry.k8s.io/kube-apiserver-amd64:v1.34.0

❯ docker run -it --rm --entrypoint="" registry.k8s.io/kube-apiserver:v1.34.0 kube-apiserver --help | grep "bootstrap"

     --enable-bootstrap-token-auth                       Enable to allow secrets of type 'bootstrap.kubernetes.io/token' in the 'kube-system' namespace to be used for TLS bootstrapping authentication.
```

In simple words, how I understand it as:

- If I set the flag `--enable-bootstrap-token-auth` to `True`, then the API server is configured to trust a list of (kubeadm) bootstrap tokens.
- These tokens needs to be stored as a `Secret` object (of a very specific type - `bootstrap.kubernetes.io/token`)  in the `kube-system` namespace.
- and once that is done, then if a joining node (actually, the kubelet running on the joining node) makes a "TLS bootstrapping authentication" request using the "bootstrap token" in the request header (`Authorization: Bearer <token>`), then the request is authenticated.

So, we have some information now about the kubeadm bootstrap token.

But you know what, `kubeadm` itself explains the role of the token much better:

```bash
❯ root@kind-control-plane:/# kubeadm token --help

This command manages bootstrap tokens. It is optional and needed only for advanced use cases.

In short, bootstrap tokens are used for establishing bidirectional trust between a client and a server.
A bootstrap token can be used when a client (for example a node that is about to join the cluster) needs
to trust the server it is talking to. Then a bootstrap token with the "signing" usage can be used.
bootstrap tokens can also function as a way to allow short-lived authentication to the API Server
(the token serves as a way for the API Server to trust the client), for example for doing the TLS Bootstrap.

What is a bootstrap token more exactly?
 - It is a Secret in the kube-system namespace of type "bootstrap.kubernetes.io/token".
 - A bootstrap token must be of the form "[a-z0-9]{6}.[a-z0-9]{16}". The former part is the public token ID,
   while the latter is the Token Secret and it must be kept private at all circumstances!
 - The name of the Secret must be named "bootstrap-token-(token-id)".

You can read more about bootstrap tokens here:
  https://kubernetes.io/docs/admin/bootstrap-tokens/
```

Also, the link at the bottom doesn't work anymore. Correct one (atleast as of writing) is - [https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/).


So, at this point we have a verbal answer for this question - `what is the role of the kubeadm token?`.

Next, I want to try is to create a manual kubeamd token (and now, from the `kubeadm --help` output, I also know what is going to be the format of a valid kubeadm token).

---

I am continuing on the Kind cluster we created above.  

The control-plane node (docker container) was created with the IP address - `172.18.0.2`.  
(leaving it here as a note, because we will use it later. And this is the future me talking, I didn't know it as I was trying things ofcourse)

And to simulate a new node, I didn’t use kind’s built-in multi-node support.  
Instead, I created a plain docker container on the same network (the docker bridge network with the name, `kind`):

```bash
❯ docker run --rm -it --name joining-node \
  --privileged \
  --network kind \
  kindest/node:latest

❯ docker container ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED             STATUS             PORTS                       NAMES
83ab08acf723   kindest/node:latest    "/usr/local/bin/entr…"   7 seconds ago       Up 6 seconds                                   joining-node
```

This new container as of now, has nothing but the basic tools installed (`kubeadm`, `kubelet`, `kubectl` and a clean filesystem, coming from the `kindest/node` image). So, as of now, no certificates, no kubeconfig.

Also, note the name of the container, `joining-node`.  
(I will use it later to exec inside the container and use it as a joining node).

---

Back to the rules we got earlier:

> What is a bootstrap token more exactly?
> - It is a Secret in the kube-system namespace of type "bootstrap.kubernetes.io/token".
> - A bootstrap token must be of the form "[a-z0-9]{6}.[a-z0-9]{16}". The former part is the public token ID,
   while the latter is the Token Secret and it must be kept private at all circumstances!
> - The name of the Secret must be named "bootstrap-token-(token-id)".


I am creating the following Secret object in the cluster (from the `kind-control-plane` node).  
Notice the token bits I added in the yaml - `token-id: pqrstu` and `token-secret: abcdef1234567890`.  
Together, these will give us the full token as `pqrstu.abcdef1234567890`.  
We will use this to make our `kubeadm join` requests.

To come up with the following template, I referred to existing tokens from other multi-node Kind cluster.  
(updated later: it is also available here - [Bootstrap Token Secret Format](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/#bootstrap-token-secret-format))

```yaml
# bootstrap-token.yaml

apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-pqrstu
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: pqrstu
  token-secret: abcdef1234567890
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  expiration: "2030-01-01T00:00:00Z"
```

Apply it now:

```bash
❯ kubectl apply -f bootstrap-token.yaml

❯ kubectl get secrets -n kube-system

NAME                     TYPE                            DATA   AGE
bootstrap-token-abcdef   bootstrap.kubernetes.io/token   6      58m

---

Now let's make some first attempts at joining from the new docker container node (`joining-node`) and see how Kubeadm behaves (I have kept the verbosity of the logs very high).

Inside the new node docker container, I started by running `kubeadm join` with no arguments:

```bash
❯ docker exec -it joining-node /bin/bash

root@83ab08acf723:/# kubeadm join --v=9

I1216 07:00:01.797859     164 join.go:423] [preflight] found NodeName empty; using OS hostname as NodeName
I1216 07:00:01.798056     164 initconfiguration.go:122] detected and using CRI socket: unix:///var/run/containerd/containerd.sock
error: discovery: Invalid value: "": bootstrapToken or file must be set
no stack trace
```

That failed immediately. Next, let's try to give it the token we created above:

```bash
root@83ab08acf723:/# kubeadm join pqrstu.abcdef1234567890 --v=9

error: [discovery.bootstrapToken.caCertHashes: Invalid value: "": using token-based discovery without caCertHashes can be unsafe. Set unsafeSkipCAVerification as true in your kubeadm config file or pass --discovery-token-unsafe-skip-ca-verification flag to continue, discovery.bootstrapToken.token: Invalid value: "": the bootstrap token is invalid, discovery.bootstrapToken.apiServerEndpoint: Invalid value: "pqrstu.abcdef1234567890": address pqrstu.abcdef1234567890: missing port in address, discovery.tlsBootstrapToken: Invalid value: "": the bootstrap token is invalid]
no stack trace
```

That produced a more informative error.
Now, we atleast know that kubeadm expects an endpoint to the API server first and then the token (maybe).

Also the logs suggested to pass `--discovery-token-unsafe-skip-ca-verification` flag to continue. Let's do that as well, to make some progress.

(and yes, I could have already looked at the `kubeadm --help` menu to understand the required format, but above test runs were intentional to understand things).

---

Now let's provide everything properly that kubeadm actually needs:

```bash
root@83ab08acf723:/# kubeadm join 172.18.0.2:6443 \
  --token="pqrstu.abcdef1234567890" \
  --discovery-token-unsafe-skip-ca-verification --v=9
```

And, this time things moved some futher.  

let's look at some important bits of the logs part by part.

```bash
230 token.go:229] [discovery] Waiting for the cluster-info ConfigMap to receive a JWS signature for token ID "pqrstu"
230 type.go:165] "Request Body" body=""
230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" 'https://172.18.0.2:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s'
 >
230 round_trippers.go:562] "HTTP Trace: Dial succeed" network="tcp" address="172.18.0.2:6443"
230 round_trippers.go:632] "Response" verb="GET" url="https://172.18.0.2:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s" status="200 OK" headers=<...>
```

From the above bits, it looks like at this point, we are able to successfully _**authenticate**_ using the manually created token we passed (look at the `status="200 OK"`).

But then after authentication, the API Server refused us to continue futher.

```bash
230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" -H "Authorization: Bearer <masked>" 'https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s'
 >

230 round_trippers.go:632] "Response" verb="GET" url="https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s" status="403 Forbidden" headers=<...>

230 token.go:249] [discovery] Retrying due to error: could not find a JWS signature in the cluster-info ConfigMap for token ID "pqrstu"

unable to fetch the kubeadm-config ConfigMap:
configmaps "kubeadm-config" is forbidden:
User "system:bootstrap:pqrstu" cannot get resource "configmaps"
in namespace "kube-system"
```

And we are also able to see where is the problem - `User "system:bootstrap:pqrstu" cannot get resource "configmaps" in namespace "kube-system"`.

So, we now know when we passed the token to the `kubeadm join` command, the API Server sees our request as coming from the user `system:bootstrap:pqrstu`.

ok, let's try to fix it now by giving it the required permissions.

We know that kubernetes uses RBAC to configure these kind of permissions on the kubernetes objects.  
(and once again, I looked at another multi-node kind cluster to see what permissions we are missing and came up with the following template).

I created a `ClusterRoleBinding` object that allowed the user (`system:bootstrap:pqrstu`) to read cluster configuration.

(AND PLEASE NOTE! This is not something I would do in a real cluster. This is purely just to move forward with this test!)

```yaml
# clusterrolebinding-bootstrap-token.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-bootstrap-read-kubeadm-config
subjects:
- kind: User
  name: system:bootstrap:pqrstu
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Apply it to the cluster:


```bash
❯ kubectl apply -f clusterrolebinding-bootstrap-token.yaml 
clusterrolebinding.rbac.authorization.k8s.io/allow-bootstrap-read-kubeadm-config created

❯ kubectl get clusterrolebinding allow-bootstrap-read-kubeadm-config -n kube-system 
NAME                                  ROLE                        AGE
allow-bootstrap-read-kubeadm-config   ClusterRole/cluster-admin   49s
```

Now, we have configured more permissions for our user `system:bootstrap:pqrstu`.  

Let's re-run the same same command:

```bash
root@83ab08acf723:/# kubeadm join 172.18.0.2:6443 \
  --token="pqrstu.abcdef1234567890" \
  --discovery-token-unsafe-skip-ca-verification --v=9
```

Voila! This time it worked successfully. The `kubeadm join` ran successfully and retured - `This node has joined the cluster`.

(The full logs are at the bottom of this post. Please please go look at them.
Those contain all the requests logged with the token in use in request's headers.  
I will try to bring them back here inline but right now the folding-y thing doesn't work properly.)

On the Kind cluster's control-plane node, I can see this new docker container node showing up:

```bash
❯ kubectl get nodes 
NAME                 STATUS     ROLES           AGE    VERSION
83ab08acf723         NotReady   <none>          108s   v1.35.0-alpha.2.488+f35f9509a69cc6
kind-control-plane   Ready      control-plane   106m   v1.34.0
```

So, at this point now.  
We've answered our second question also.   
i.e, `can I try to create a kubeadm token manually and use that to join an existing Kubernetes cluster successfully?`.  
Yes, I can! We saw it above!

But wait, what next now?

We saw the following bit in our output:

```bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

I have more questions now.
- What is the "Certificate signing request"? And what "response" we received?
- The Kubelet was informed of the new secure connection details. How?

Let's try to answer them now.

So, back to the control-plane node, let's check the following:

```bash
❯ kubectl get certificatesigningrequest -A

NAME        AGE     SIGNERNAME                                    REQUESTOR                        REQUESTEDDURATION   CONDITION
csr-c5j6z   7m46s   kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:pqrstu          <none>              Approved,Issued
csr-qzbvk   112m    kubernetes.io/kube-apiserver-client-kubelet   system:node:kind-control-plane   <none>              Approved,Issued
```

OK, so, we see two `CertificateSigningRequest` (CSR) objects.  
One of them was requested by the `system:node:kind-control-plane` user (our kind-control-plane node).  
And the other one was requested by us, through the user `system:bootstrap:pqrstu` from the docker container node (`joining-node`).  
And both are in condition `Approved` and `Issued`.  

Let's also see the body of the CSR object created by our request.

```yaml
❯ kubectl get certificatesigningrequest csr-c5j6z -o yaml

apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  ...
spec:
  groups:
  - system:bootstrappers
  - system:authenticated
  request: LS0tLS1CRUdJTi****LS0K
  signerName: kubernetes.io/kube-apiserver-client-kubelet
  usages:
  - digital signature
  - client auth
  username: system:bootstrap:pqrstu
status:
  certificate: LS0tLS1CRUdJTiBDR****LS0K
  conditions:
  - ...
    message: Auto approving kubelet client certificate after SubjectAccessReview.
    reason: AutoApproved
    status: "True"
    type: Approved
```

From the object definition, I understand that the user `system:bootstrap:pqrstu`, made this CSR request for purposes, `usages: {digital signature, client auth}`.  
And that is `AutoApproved` (after some process called `SubjectAccessReview` which I am not getting into, in this blog, but I know that is important).

Nice. But what did we receive as response?

I see, we got back a `certificate` and `Approved` (with `status: "True"`) in the CSR object's status section.   
So, that means the response we got back is the CSR getting approved. (maybe, I'm still not 100% sure about which response we're talking).

ok, next we also saw the part - "Kubelet was informed of the new secure connection details".  
I want to understand how Kubelet was informed.

I do see some relevant logs (truncated to just show the relevant bits):

```bash
[kubelet-start] writing CA certificate at /etc/kubernetes/pki/ca.crt

[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
```

Let's check if we see anything new inside our docker container node we joined from (`joining-node`).

```bash
root@83ab08acf723:/# tree /etc/kubernetes/
/etc/kubernetes/
|-- kubelet.conf
|-- manifests
`-- pki
    `-- ca.crt

3 directories, 2 files
```

We do.  
We now have a new directory called `/etcd/kubernetes/` which contains a `kubelet.conf` file as well as `pki/ca.crt`.  
(same as what the logs pointed us.)

Let's see the contents of the `kubelet.conf` file:

```yaml
root@0ecd1b55abc8:/# cat /etc/kubernetes/kubelet.conf 

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUd****LS0tLS0K
    server: https://kind-control-plane:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
users:
- name: default-auth
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

The `certificate-authority-data` field stores the public certificate of Cluster's CA (Which is coming from the control-plane of the cluster).
If you decode the value (using `echo "LS0tLS1CRUd****LS0tLS0K" | base64 -d`, it will match the contents of the `/etc/kubernetes/pki/ca.crt` on the kind-control-plane node.

We also see it points to the location of the kubelet's certificates & keys - `/var/lib/kubelet/pki/`.

As I understand (from reading docs):
the kubelet generates its own certificates/keys locally (`/var/lib/kubelet/pki/Kubelet.crt/key`) and then sent the CSR to the API Server.  
The control-plane (controller-manager?) then sign that CSR using the cluster CA keys (which we see from the field - `certificate-authority-data`).  
And then signed certificate is written into `kubelet.conf` (to be precise here - `/var/lib/kubelet/pki/kubelet-client-current.pem`, which comes from the `status.certificate:` field of the CSR Object).

The full tree of `/var/lib/kubelet/pki/` looks like:

```bash
root@fd5e81a7604b:/# tree /var/lib/kubelet/pki/
/var/lib/kubelet/pki/
|-- kubelet-client-2025-12-16-07-35-31.pem
|-- kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2025-12-16-07-35-31.pem
|-- kubelet.crt
`-- kubelet.key

1 directory, 4 files
```

(Note: The part "kubelet generate a private key and a CSR for submission to a cluster-level certificate signing process." was originally proposed as part of this design proposal - [Kubelet TLS bootstrap](https://github.com/kubernetes/design-proposals-archive/blob/main/cluster-lifecycle/kubelet-tls-bootstrap.md))


So, after this point onwards, our (`joining-node`) node aka the Kubelet has its own set of signed certificates.  
And so, the kubeadm bearer token is no longer required.  
And all further interactions with the control-plane will use these certificates via mTLS.

Also, note that - not just certificates, a lot more stuff was also added to the filesystem of the (`joining-node`) node.  
I don't know details about every single listed item, but here's the full tree:

```bash
root@0ecd1b55abc8:/# tree /var/lib/kubelet/
/var/lib/kubelet/
|-- allocated_pods_state
|-- checkpoints
|-- config.yaml
|-- cpu_manager_state
|-- device-plugins
|   `-- kubelet.sock
|-- dra_manager_state
|-- instance-config.yaml
|-- kubeadm-flags.env
|-- memory_manager_state
|-- pki
|   |-- kubelet-client-2025-12-16-10-18-54.pem
|   |-- kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2025-12-16-10-18-54.pem
|   |-- kubelet.crt
|   `-- kubelet.key
|-- plugins
|-- plugins_registry
|-- pod-resources
|   `-- kubelet.sock
`-- pods
    |-- 4683e97e-5735-459f-b965-24a5ffd2f63e
    |   |-- plugins
    |   |   `-- kubernetes.io~empty-dir
    |   |       `-- wrapped_kube-api-access-7v75d
    |   |           `-- ready
    |   `-- volumes
    |       `-- kubernetes.io~projected
    |           `-- kube-api-access-7v75d
    |               |-- ca.crt -> ..data/ca.crt
    |               |-- namespace -> ..data/namespace
    |               `-- token -> ..data/token
    `-- d6d5a0a6-4c4a-4cc0-984f-a63129fbefca
        |-- plugins
        |   `-- kubernetes.io~empty-dir
        |       |-- wrapped_kube-api-access-c4l5m
        |       |   `-- ready
        |       `-- wrapped_kube-proxy
        |           `-- ready
        `-- volumes
            |-- kubernetes.io~configmap
            |   `-- kube-proxy
            |       |-- config.conf -> ..data/config.conf
            |       `-- kubeconfig.conf -> ..data/kubeconfig.conf
            `-- kubernetes.io~projected
                `-- kube-api-access-c4l5m
                    |-- ca.crt -> ..data/ca.crt
                    |-- namespace -> ..data/namespace
                    `-- token -> ..data/token

25 directories, 24 files
```

OK, I already feel I learnt quite a bit.

Yet, our very first question is still not answered.  
i.e., why there's a symmetric token used by kubeadm?

I was actually having a chat with the creator of Kubeadm himself, Lucas Käldström (yes, the same person who wrote the above linked thesis).  
What I learnt is - even though it's a symmetric and shared string, the token itself has two parts, where the first part (`token-id`) is supposed to be public and the second part (`token-secret`) is supposed to be private.  
And as the name suggests the second private part of the token is supposed to be kept private (obviously.)  
Becuase, Kubeadm tokens are to be used for establishing bidirectional trust between the client (in our case, `joining-node`) and the server (the control-plane, `api-server`).  
- For the client (`joining-node`) to establish trust to the server (the control-plane, `api-server`), we saw the first part (`token-id`) of the token is used (`system:bootstrap:pqrstu` and the matching secret and clusterrolebinding).  
- For the server (the control-plane, `api-server`) to establish trust to the client (`joining-node`), the entire shared token (both `token-id` and `token-secret`) can be used.  

  If you look at the full kubeadm join logs again, one of the early steps you will see is - `attempting to download the KubeletConfiguration from ConfigMap "kubelet-config"`.
  This token (stored as the Secret object in the control-plane aka the server side) can be used to sign the ConfigMap ("kubelet-config") and then on the client side (`joining-node`), the received signed ConfigMap can then be authenticated by using the same shared token.  
  The process is explained here briefly - [ConfigMap Signing](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/#configmap-signing), but the important part is - `You can verify the JWS (signature) using the HS256 scheme (HMAC-SHA256) with the full token (e.g. 07401b.f395accd246ae52d) as the shared secret.`

So, even though it's a symmetric and a shared token, but it has similarities to the assymetric key pairs with its public and private parts used for separate purposes.  
There's also this design proposal called, [bootstrap discovery](https://github.com/kubernetes/design-proposals-archive/blob/main/cluster-lifecycle/bootstrap-discovery.md), which discusses the flow we see in our logs.


Finally, I will end this post with this diagram which atleast for me, summarises nicely, the process we followed from start to end so far:


```pgsql
New node
  │
  │ kubeadm join
  │
  ▼
API Server (unauthenticated)
  │
  │ token-based auth
  ▼
CSR created
  │
  │ CertificateSigningRequest
  ▼
Controller approves CSR
  │
  │ signed by CA
  ▼
kubelet gets client cert
  │
  │ mTLS from now on
  ▼
FULLY TRUSTED NODE
```


With that, thank you for reading so far. And I hope you also got to learn a few new things. o/

---
---

PS: Even though the node was able to join the control-plane, but it wasn't really in a `READY` state (and I left it there, didn't troubleshoot it further).   

The Kubelet logs read something like:

```
failed to mount rootfs component: mount source "overlay" ... err: invalid argument
```


---


<details markdown="1">
  <summary>Please see the full raw kubeadm logs here. (click to expand)</summary>
{% raw %}
```bash
root@83ab08acf723:/# kubeadm join 172.18.0.2:6443   --token="pqrstu.abcdef1234567890"   --discovery-token-unsafe-skip-ca-verification --v=9
I1215 14:50:15.160788     230 join.go:423] [preflight] found NodeName empty; using OS hostname as NodeName
I1215 14:50:15.161142     230 initconfiguration.go:122] detected and using CRI socket: unix:///var/run/containerd/containerd.sock
[preflight] Running pre-flight checks
I1215 14:50:15.161266     230 preflight.go:93] [preflight] Running general checks
I1215 14:50:15.161321     230 checks.go:315] validating the existence of file /etc/kubernetes/kubelet.conf
I1215 14:50:15.161355     230 checks.go:315] validating the existence of file /etc/kubernetes/bootstrap-kubelet.conf
I1215 14:50:15.161382     230 checks.go:89] validating the container runtime
I1215 14:50:15.167560     230 checks.go:120] validating the container runtime version compatibility
I1215 14:50:15.170013     230 checks.go:685] validating whether swap is enabled or not
	[WARNING Swap]: swap is supported for cgroup v2 only. The kubelet must be properly configured to use swap. Please refer to https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory, or disable swap on the node
I1215 14:50:15.170160     230 checks.go:405] validating the presence of executable losetup
I1215 14:50:15.170232     230 checks.go:405] validating the presence of executable mount
I1215 14:50:15.170257     230 checks.go:405] validating the presence of executable cp
I1215 14:50:15.170278     230 checks.go:551] running system verification checks
I1215 14:50:15.231856     230 checks.go:436] checking whether the given node name is valid and reachable using net.LookupHost
I1215 14:50:15.232022     230 checks.go:651] validating kubelet version
I1215 14:50:15.276735     230 checks.go:165] validating if the "kubelet" service is enabled and active
I1215 14:50:15.294376     230 checks.go:238] validating availability of port 10250
I1215 14:50:15.294632     230 checks.go:315] validating the existence of file /etc/kubernetes/pki/ca.crt
I1215 14:50:15.294661     230 checks.go:465] validating if the connectivity type is via proxy or direct
I1215 14:50:15.294728     230 checks.go:364] validating the contents of file /proc/sys/net/ipv4/ip_forward
I1215 14:50:15.294770     230 join.go:553] [preflight] Discovering cluster-info
I1215 14:50:15.294792     230 token.go:71] [discovery] Created cluster-info discovery client, requesting info from "172.18.0.2:6443"
I1215 14:50:15.294942     230 envvar.go:172] "Feature gate default state" feature="InOrderInformersBatchProcess" enabled=true
I1215 14:50:15.294959     230 envvar.go:172] "Feature gate default state" feature="InformerResourceVersion" enabled=true
I1215 14:50:15.294967     230 envvar.go:172] "Feature gate default state" feature="WatchListClient" enabled=false
I1215 14:50:15.294976     230 envvar.go:172] "Feature gate default state" feature="ClientsAllowCBOR" enabled=false
I1215 14:50:15.294984     230 envvar.go:172] "Feature gate default state" feature="ClientsPreferCBOR" enabled=false
I1215 14:50:15.294991     230 envvar.go:172] "Feature gate default state" feature="InOrderInformers" enabled=true
I1215 14:50:15.295371     230 token.go:229] [discovery] Waiting for the cluster-info ConfigMap to receive a JWS signature for token ID "pqrstu"
I1215 14:50:15.295452     230 type.go:165] "Request Body" body=""
I1215 14:50:15.295539     230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" 'https://172.18.0.2:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s'
 >
I1215 14:50:15.295931     230 round_trippers.go:562] "HTTP Trace: Dial succeed" network="tcp" address="172.18.0.2:6443"
I1215 14:50:15.302680     230 round_trippers.go:632] "Response" verb="GET" url="https://172.18.0.2:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s" status="200 OK" headers=<
	Audit-Id: 6c3035a3-0824-415c-8e5d-0db271cdb29f
	Cache-Control: no-cache, private
	Content-Length: 2102
	Content-Type: application/vnd.kubernetes.protobuf
	Date: Mon, 15 Dec 2025 14:50:15 GMT
	X-Kubernetes-Pf-Flowschema-Uid: f54c5d84-c2d8-4cbb-8801-07adb48ce73d
	X-Kubernetes-Pf-Prioritylevel-Uid: 7df0c0a7-2f8a-40ca-9625-0b6635feccd2
 > milliseconds=7 dnsLookupMilliseconds=0 dialMilliseconds=0 tlsHandshakeMilliseconds=4 serverProcessingMilliseconds=2
I1215 14:50:15.302857     230 type.go:165] "Response Body" body=<
	00000000  6b 38 73 00 0a 0f 0a 02  76 31 12 09 43 6f 6e 66  |k8s.....v1..Conf|
	00000010  69 67 4d 61 70 12 9a 10  0a 99 02 0a 0c 63 6c 75  |igMap........clu|
	00000020  73 74 65 72 2d 69 6e 66  6f 12 00 1a 0b 6b 75 62  |ster-info....kub|
	00000030  65 2d 70 75 62 6c 69 63  22 00 2a 24 38 34 37 38  |e-public".*$8478|
	00000040  65 64 38 35 2d 62 61 66  39 2d 34 63 63 61 2d 38  |ed85-baf9-4cca-8|
	00000050  31 65 32 2d 37 35 36 66  62 39 61 61 65 62 62 61  |1e2-756fb9aaebba|
	00000060  32 04 38 34 33 35 38 00  42 08 08 d7 da ff c9 06  |2.84358.B.......|
	00000070  10 00 8a 01 54 0a 07 6b  75 62 65 61 64 6d 12 06  |....T..kubeadm..|
	00000080  55 70 64 61 74 65 1a 02  76 31 22 08 08 d7 da ff  |Update..v1".....|
	00000090  c9 06 10 00 32 08 46 69  65 6c 64 73 56 31 3a 27  |....2.FieldsV1:'|
	000000a0  0a 25 7b 22 66 3a 64 61  74 61 22 3a 7b 22 2e 22  |.%{"f:data":{"."|
	000000b0  3a 7b 7d 2c 22 66 3a 6b  75 62 65 63 6f 6e 66 69  |:{},"f:kubeconfi|
	000000c0  67 22 3a 7b 7d 7d 7d 42  00 8a 01 68 0a 17 6b 75  |g":{}}}B...h..ku|
	000000d0  62 65 2d 63 6f 6e 74 72  6f 6c 6c 65 72 2d 6d 61  |be-controller-ma|
	000000e0  6e 61 67 65 72 12 06 55  70 64 61 74 65 1a 02 76  |nager..Update..v|
	000000f0  31 22 08 08 a8 87 80 ca  06 10 00 32 08 46 69 65  |1".........2.Fie|
	00000100  6c 64 73 56 31 3a 2b 0a  29 7b 22 66 3a 64 61 74  |ldsV1:+.){"f:dat|
	00000110  61 22 3a 7b 22 66 3a 6a  77 73 2d 6b 75 62 65 63  |a":{"f:jws-kubec|
	00000120  6f 6e 66 69 67 2d 70 71  72 73 74 75 22 3a 7b 7d  |onfig-pqrstu":{}|
	00000130  7d 7d 42 00 12 6e 0a 15  6a 77 73 2d 6b 75 62 65  |}}B..n..jws-kube|
	00000140  63 6f 6e 66 69 67 2d 70  71 72 73 74 75 12 55 65  |config-pqrstu.Ue|
	00000150  79 4a 68 62 47 63 69 4f  69 4a 49 55 7a 49 31 4e  |yJhbGciOiJIUzI1N|
	00000160  69 49 73 49 6d 74 70 5a  43 49 36 49 6e 42 78 63  |iIsImtpZCI6InBxc|
	00000170  6e 4e 30 64 53 4a 39 2e  2e 75 5f 4d 74 56 4f 42  |nN0dSJ9..u_MtVOB|
	00000180  69 68 65 4e 5a 37 5a 57  31 5f 5a 36 52 6d 67 78  |iheNZ7ZW1_Z6Rmgx|
	00000190  35 50 41 5f 69 76 6b 77  31 2d 62 46 53 35 49 50  |5PA_ivkw1-bFS5IP|
	000001a0  71 33 34 55 12 8b 0d 0a  0a 6b 75 62 65 63 6f 6e  |q34U.....kubecon|
	000001b0  66 69 67 12 fc 0c 61 70  69 56 65 72 73 69 6f 6e  |fig...apiVersion|
	000001c0  3a 20 76 31 0a 63 6c 75  73 74 65 72 73 3a 0a 2d  |: v1.clusters:.-|
	000001d0  20 63 6c 75 73 74 65 72  3a 0a 20 20 20 20 63 65  | cluster:.    ce|
	000001e0  72 74 69 66 69 63 61 74  65 2d 61 75 74 68 6f 72  |rtificate-author|
	000001f0  69 74 79 2d 64 61 74 61  3a 20 4c 53 30 74 4c 53  |ity-data: LS0tLS|
	00000200  31 43 52 55 64 4a 54 69  42 44 52 56 4a 55 53 55  |1CRUdJTiBDRVJUSU|
	00000210  5a 4a 51 30 46 55 52 53  30 74 4c 53 30 74 43 6b  |ZJQ0FURS0tLS0tCk|
	00000220  31 4a 53 55 52 43 56 45  4e 44 51 57 55 79 5a 30  |1JSURCVENDQWUyZ0|
	00000230  46 33 53 55 4a 42 5a 30  6c 4a 55 46 70 36 4e 31  |F3SUJBZ0lJUFp6N1|
	00000240  5a 44 52 7a 6c 4b 64 6b  31 33 52 46 46 5a 53 6b  |ZDRzlKdk13RFFZSk|
	00000250  74 76 57 6b 6c 6f 64 6d  4e 4f 51 56 46 46 54 45  |tvWklodmNOQVFFTE|
	00000260  4a 52 51 58 64 47 56 45  56 55 54 55 4a 46 52 30  |JRQXdGVEVUTUJFR0|
	00000270  45 78 56 55 55 4b 51 58  68 4e 53 32 45 7a 56 6d  |ExVUUKQXhNS2EzVm|
	00000280  6c 61 57 45 70 31 57 6c  68 53 62 47 4e 36 51 57  |laWEp1WlhSbGN6QW|
	00000290  56 47 64 7a 42 35 54 6c  52 46 65 55 31 55 56 58  |VGdzB5TlRFeU1UVX|
	000002a0  68 4e 56 45 45 30 54 56  52 57 59 55 5a 33 4d 48  |hNVEE0TVRWYUZ3MH|
	000002b0  70 4f 56 45 56 35 54 56  52 4e 65 45 31 55 52 58  |pOVEV5TVRNeE1URX|
	000002c0  70 4e 56 46 5a 68 54 55  4a 56 65 41 70 46 65 6b  |pNVFZhTUJVeApFek|
	000002d0  46 53 51 6d 64 4f 56 6b  4a 42 54 56 52 44 62 58  |FSQmdOVkJBTVRDbX|
	000002e0  51 78 57 57 31 57 65 57  4a 74 56 6a 42 61 57 45  |QxWW1WeWJtVjBaWE|
	000002f0  31 33 5a 32 64 46 61 55  31 42 4d 45 64 44 55 33  |13Z2dFaU1BMEdDU3|
	00000300  46 48 55 30 6c 69 4d 30  52 52 52 55 4a 42 55 56  |FHU0liM0RRRUJBUV|
	00000310  56 42 51 54 52 4a 51 6b  52 33 51 58 64 6e 5a 30  |VBQTRJQkR3QXdnZ0|
	00000320  56 4c 43 6b 46 76 53 55  4a 42 55 55 52 34 4d 6a  |VLCkFvSUJBUUR4Mj|
	00000330  56 79 64 55 35 61 54 6e  42 70 62 6a 68 30 51 54  |VydU5aTnBpbjh0QT|
	00000340  46 4d 52 6d 39 71 4e 31  6c 74 4e 44 56 6b 4d 30  |FMRm9qN1ltNDVkM0|
	00000350  4e 30 65 56 42 71 57 47  73 30 53 6d 4a 31 54 46  |N0eVBqWGs0SmJ1TF|
	00000360  70 53 56 55 55 33 61 6e  6c 59 53 7a 64 6d 4d 6e  |pSVUU3anlYSzdmMn|
	00000370  70 49 53 44 45 35 4d 46  59 4b 54 58 42 36 4e 31  |pISDE5MFYKTXB6N1|
	00000380  46 59 62 7a 64 32 53 31  70 6f 65 46 6c 70 55 7a  |FYbzd2S1poeFlpUz|
	00000390  64 46 61 46 52 4d 65 56  4e 6d 4f 47 68 6f 53 32  |dFaFRMeVNmOGhoS2|
	000003a0  59 34 56 32 55 34 5a 6b  6c 58 55 6b 46 45 4f 56  |Y4V2U4ZklXUkFEOV|
	000003b0  6c 55 56 57 70 61 59 6c  4a 6a 63 6a 4e 6f 52 46  |lUVWpaYlJjcjNoRF|
	000003c0  6f 72 57 44 68 6c 62 47  70 78 52 48 41 31 51 51  |orWDhlbGpxRHA1QQ|
	000003d0  70 52 65 44 4e 34 62 32  52 55 4d 6c 68 4c 4e 31  |pReDN4b2RUMlhLN1|
	000003e0  56 72 54 6e 68 50 64 6d  39 36 5a 58 68 42 4c 32  |VrTnhPdm96ZXhBL2|
	000003f0  74 53 4f 58 6c 78 65 6d  4a 72 57 56 70 7a 61 30  |tSOXlxemJrWVpza0|
	00000400  46 42 59 58 46 5a 51 54  4e 71 52 7a 42 4a 63 56  |FBYXFZQTNqRzBJcV|
	00000410  4e 46 51 58 68 4c 52 56  68 4c 57 6e 64 4d 4d 6d  |NFQXhLRVhLWndMMm|
	00000420  77 77 59 32 45 77 43 6e  70 4b 52 6a 6c 6e 62 54  |wwY2EwCnpKRjlnbT|
	00000430  4e 33 64 33 68 4f 55 47  64 49 53 57 4e 73 4e 57  |N3d3hOUGdISWNsNW|
	00000440  68 68 63 45 56 46 4e 54  56 36 63 30 67 77 65 47  |hhcEVFNTV6c0gweG|
	00000450  68 76 57 58 64 4b 5a 32  78 53 4f 57 31 6d 5a 6c  |hvWXdKZ2xSOW1mZl|
	00000460  64 4e 4e 7a 68 6b 52 32  4d 78 55 33 68 79 52 57  |dNNzhkR2MxU3hyRW|
	00000470  46 30 65 6d 5a 52 4f 44  46 72 59 55 77 4b 56 6b  |F0emZRODFrYUwKVk|
	00000480  52 50 5a 30 56 6b 62 58  4e 4b 65 48 52 4a 55 32  |RPZ0VkbXNKeHRJU2|
	00000490  45 31 53 6d 74 43 54 58  56 52 5a 7a 56 52 57 57  |E1SmtCTXVRZzVRWW|
	000004a0  39 43 51 57 4e 78 55 30  6b 31 64 33 52 30 4e 47  |9CQWNxU0k1d3R0NG|
	000004b0  6b 34 61 6c 6f 76 64 6e  70 45 64 6d 56 36 4e 48  |k4alovdnpEdmV6NH|
	000004c0  67 33 55 45 64 77 63 56  70 5a 56 54 5a 4c 54 33  |g3UEdwcVpZVTZLT3|
	000004d0  6c 48 5a 67 70 75 64 46  42 4a 55 55 49 33 64 55  |lHZgpudFBJUUI3dU|
	000004e0  46 6f 52 57 78 4d 65 46  52 53 4b 7a 4a 68 57 54  |FoRWxMeFRSKzJhWT|
	000004f0  63 7a 53 55 39 73 63 56  6b 76 51 57 64 4e 51 6b  |czSU9scVkvQWdNQk|
	00000500  46 42 52 32 70 58 56 45  4a 59 54 55 45 30 52 30  |FBR2pXVEJYTUE0R0|
	00000510  45 78 56 57 52 45 64 30  56 43 4c 33 64 52 52 55  |ExVWREd0VCL3dRRU|
	00000520  46 33 53 55 4e 77 52 45  46 51 43 6b 4a 6e 54 6c  |F3SUNwREFQCkJnTl|
	00000530  5a 49 55 6b 31 43 51 57  59 34 52 55 4a 55 51 55  |ZIUk1CQWY4RUJUQU|
	00000540  52 42 55 55 67 76 54 55  49 77 52 30 45 78 56 57  |RBUUgvTUIwR0ExVW|
	00000550  52 45 5a 31 46 58 51 6b  4a 53 55 79 39 56 4e 44  |REZ1FXQkJSUy9VND|
	00000560  4e 73 59 6a 51 32 62 45  70 74 52 48 6b 78 57 56  |NsYjQ2bEptRHkxWV|
	00000570  42 7a 56 6e 64 43 4d 31  4a 44 55 44 52 45 51 56  |BzVndCM1JDUDREQV|
	00000580  59 4b 51 6d 64 4f 56 6b  68 53 52 55 56 45 61 6b  |YKQmdOVkhSRUVEak|
	00000590  46 4e 5a 32 64 77 63 6d  52 58 53 6d 78 6a 62 54  |FNZ2dwcmRXSmxjbT|
	000005a0  56 73 5a 45 64 57 65 6b  31 42 4d 45 64 44 55 33  |VsZEdWek1BMEdDU3|
	000005b0  46 48 55 30 6c 69 4d 30  52 52 52 55 4a 44 64 31  |FHU0liM0RRRUJDd1|
	000005c0  56 42 51 54 52 4a 51 6b  46 52 51 33 56 34 64 31  |VBQTRJQkFRQ3V4d1|
	000005d0  46 4a 54 6a 64 6f 61 41  70 59 59 6d 52 4c 63 30  |FJTjdoaApYYmRLc0|
	000005e0  39 72 56 33 42 42 52 46  68 75 59 57 35 4f 57 48  |9rV3BBRFhuYW5OWH|
	000005f0  5a 4b 63 45 52 6b 54 57  38 35 61 47 38 79 54 58  |ZKcERkTW85aG8yTX|
	00000600  70 77 63 55 78 30 4f 58  46 6b 61 55 52 61 59 6b  |pwcUx0OXFkaURaYk|
	00000610  78 76 63 44 52 44 59 6e  67 72 63 30 70 53 59 7a  |xvcDRDYngrc0pSYz|
	00000620  6c 6d 62 33 4e 59 52 7a  56 58 61 6a 46 56 43 6b  |lmb3NYRzVXajFVCk|
	00000630  46 79 62 6e 6c 6a 5a 6b  6c 34 62 6b 5a 56 4d 57  |FybnljZkl4bkZVMW|
	00000640  74 70 53 48 4e 6e 64 32  55 35 4e 47 35 43 59 55  |tpSHNnd2U5NG5CYU|
	00000650  64 6a 59 56 4e 44 63 58  64 54 62 55 56 49 65 44  |djYVNDcXdTbUVIeD|
	00000660  52 73 4d 56 5a 36 53 69  74 6c 64 30 35 57 62 32  |RsMVZ6Sitld05Wb2|
	00000670  34 34 63 6e 42 75 55 30  74 59 65 47 46 52 55 33  |44cnBuU0tYeGFRU3|
	00000680  41 30 4e 6d 55 4b 4b 30  64 69 64 55 31 34 55 48  |A0NmUKK0didU14UH|
	00000690  64 56 5a 46 6c 69 64 6a  5a 32 57 44 4a 43 64 7a  |dVZFlidjZ2WDJCdz|
	000006a0  56 70 64 33 64 43 61 56  42 32 4f 57 78 43 4f 44  |Vpd3dCaVB2OWxCOD|
	000006b0  45 7a 54 6e 4a 6e 61 31  52 5a 4b 30 46 70 65 55  |EzTnJna1RZK0FpeU|
	000006c0  70 47 61 33 64 79 62 6a  46 51 53 6d 4e 55 64 6d  |pGa3dybjFQSmNUdm|
	000006d0  64 74 64 55 4a 4e 53 6b  34 7a 52 77 70 79 63 45  |dtdUJNSk4zRwpycE|
	000006e0  74 4d 55 48 6b 32 55 7a  4a 7a 4f 44 49 35 62 54  |tMUHk2UzJzODI5bT|
	000006f0  64 57 63 45 77 7a 4f 55  4d 30 4d 47 74 54 62 45  |dWcEwzOUM0MGtTbE|
	00000700  39 42 59 6d 78 4a 52 56  70 50 53 55 56 32 63 6d  |9BYmxJRVpPSUV2cm|
	00000710  46 44 65 56 68 58 54 32  68 55 62 47 56 56 56 6c  |FDeVhXT2hUbGVVVl|
	00000720  56 4c 5a 48 42 50 4e 56  5a 34 55 30 67 76 5a 56  |VLZHBPNVZ4U0gvZV|
	00000730  46 45 43 6c 64 48 57 6e  46 76 53 6a 42 35 57 56  |FECldHWnFvSjB5WV|
	00000740  4e 52 54 47 74 48 54 43  38 78 52 79 39 71 52 32  |NRTGtHTC8xRy9qR2|
	00000750  56 6c 51 57 68 4b 62 6e  56 72 4f 56 70 73 4e 54  |VlQWhKbnVrOVpsNT|
	00000760  63 79 4e 6a 63 77 62 48  4e 71 56 48 46 4d 51 55  |cyNjcwbHNqVHFMQU|
	00000770  35 68 55 69 74 58 53 30  64 47 64 7a 46 4a 4d 6e  |5hUitXS0dGdzFJMn|
	00000780  52 7a 57 56 5a 49 54 45  38 4b 62 57 35 4c 54 6e  |RzWVZITE8KbW5LTn|
	00000790  42 49 61 79 74 72 56 56  4a 34 43 69 30 74 4c 53  |BIaytrVVJ4Ci0tLS|
	000007a0  30 74 52 55 35 45 49 45  4e 46 55 6c 52 4a 52 6b  |0tRU5EIENFUlRJRk|
	000007b0  6c 44 51 56 52 46 4c 53  30 74 4c 53 30 4b 0a 20  |lDQVRFLS0tLS0K. |
	000007c0  20 20 20 73 65 72 76 65  72 3a 20 68 74 74 70 73  |   server: https|
	000007d0  3a 2f 2f 6b 69 6e 64 2d  63 6f 6e 74 72 6f 6c 2d  |://kind-control-|
	000007e0  70 6c 61 6e 65 3a 36 34  34 33 0a 20 20 6e 61 6d  |plane:6443.  nam|
	000007f0  65 3a 20 22 22 0a 63 6f  6e 74 65 78 74 73 3a 20  |e: "".contexts: |
	00000800  6e 75 6c 6c 0a 63 75 72  72 65 6e 74 2d 63 6f 6e  |null.current-con|
	00000810  74 65 78 74 3a 20 22 22  0a 6b 69 6e 64 [truncated 178 chars]
 >
I1215 14:50:15.303934     230 token.go:113] [discovery] Cluster info signature and contents are valid and no TLS pinning was specified, will use API Server "172.18.0.2:6443"
I1215 14:50:15.303958     230 discovery.go:53] [discovery] Using provided TLSBootstrapToken as authentication credentials for the join process
I1215 14:50:15.303971     230 join.go:567] [preflight] Fetching init configuration
I1215 14:50:15.303980     230 join.go:681] [preflight] Retrieving KubeConfig objects
[preflight] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[preflight] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
I1215 14:50:15.304427     230 type.go:165] "Request Body" body=""
I1215 14:50:15.304525     230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" -H "Authorization: Bearer <masked>" 'https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s'
 >
I1215 14:50:15.305280     230 round_trippers.go:547] "HTTP Trace: DNS Lookup resolved" host="kind-control-plane" address=[{"IP":"172.18.0.2","Zone":""},{"IP":"fc00:f853:ccd:e793::2","Zone":""}]
I1215 14:50:15.305519     230 round_trippers.go:562] "HTTP Trace: Dial succeed" network="tcp" address="172.18.0.2:6443"
I1215 14:50:15.312270     230 round_trippers.go:632] "Response" verb="GET" url="https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s" status="200 OK" headers=<
	Audit-Id: dc851cb8-00d2-499a-9d93-791b645d3175
	Cache-Control: no-cache, private
	Content-Length: 935
	Content-Type: application/vnd.kubernetes.protobuf
	Date: Mon, 15 Dec 2025 14:50:15 GMT
	X-Kubernetes-Pf-Flowschema-Uid: f54c5d84-c2d8-4cbb-8801-07adb48ce73d
	X-Kubernetes-Pf-Prioritylevel-Uid: 7df0c0a7-2f8a-40ca-9625-0b6635feccd2
 > milliseconds=7 dnsLookupMilliseconds=0 dialMilliseconds=0 tlsHandshakeMilliseconds=4 serverProcessingMilliseconds=2
I1215 14:50:15.312385     230 type.go:165] "Response Body" body=<
	00000000  6b 38 73 00 0a 0f 0a 02  76 31 12 09 43 6f 6e 66  |k8s.....v1..Conf|
	00000010  69 67 4d 61 70 12 8b 07  0a b9 01 0a 0e 6b 75 62  |igMap........kub|
	00000020  65 61 64 6d 2d 63 6f 6e  66 69 67 12 00 1a 0b 6b  |eadm-config....k|
	00000030  75 62 65 2d 73 79 73 74  65 6d 22 00 2a 24 38 66  |ube-system".*$8f|
	00000040  37 61 38 66 35 34 2d 37  39 62 65 2d 34 34 61 35  |7a8f54-79be-44a5|
	00000050  2d 61 63 35 63 2d 34 34  32 34 64 33 66 37 39 39  |-ac5c-4424d3f799|
	00000060  30 39 32 03 32 30 37 38  00 42 08 08 d7 da ff c9  |092.2078.B......|
	00000070  06 10 00 8a 01 5e 0a 07  6b 75 62 65 61 64 6d 12  |.....^..kubeadm.|
	00000080  06 55 70 64 61 74 65 1a  02 76 31 22 08 08 d7 da  |.Update..v1"....|
	00000090  ff c9 06 10 00 32 08 46  69 65 6c 64 73 56 31 3a  |.....2.FieldsV1:|
	000000a0  31 0a 2f 7b 22 66 3a 64  61 74 61 22 3a 7b 22 2e  |1./{"f:data":{".|
	000000b0  22 3a 7b 7d 2c 22 66 3a  43 6c 75 73 74 65 72 43  |":{},"f:ClusterC|
	000000c0  6f 6e 66 69 67 75 72 61  74 69 6f 6e 22 3a 7b 7d  |onfiguration":{}|
	000000d0  7d 7d 42 00 12 cc 05 0a  14 43 6c 75 73 74 65 72  |}}B......Cluster|
	000000e0  43 6f 6e 66 69 67 75 72  61 74 69 6f 6e 12 b3 05  |Configuration...|
	000000f0  61 70 69 53 65 72 76 65  72 3a 0a 20 20 63 65 72  |apiServer:.  cer|
	00000100  74 53 41 4e 73 3a 0a 20  20 2d 20 6c 6f 63 61 6c  |tSANs:.  - local|
	00000110  68 6f 73 74 0a 20 20 2d  20 31 32 37 2e 30 2e 30  |host.  - 127.0.0|
	00000120  2e 31 0a 20 20 65 78 74  72 61 41 72 67 73 3a 0a  |.1.  extraArgs:.|
	00000130  20 20 2d 20 6e 61 6d 65  3a 20 72 75 6e 74 69 6d  |  - name: runtim|
	00000140  65 2d 63 6f 6e 66 69 67  0a 20 20 20 20 76 61 6c  |e-config.    val|
	00000150  75 65 3a 20 22 22 0a 61  70 69 56 65 72 73 69 6f  |ue: "".apiVersio|
	00000160  6e 3a 20 6b 75 62 65 61  64 6d 2e 6b 38 73 2e 69  |n: kubeadm.k8s.i|
	00000170  6f 2f 76 31 62 65 74 61  34 0a 63 61 43 65 72 74  |o/v1beta4.caCert|
	00000180  69 66 69 63 61 74 65 56  61 6c 69 64 69 74 79 50  |ificateValidityP|
	00000190  65 72 69 6f 64 3a 20 38  37 36 30 30 68 30 6d 30  |eriod: 87600h0m0|
	000001a0  73 0a 63 65 72 74 69 66  69 63 61 74 65 56 61 6c  |s.certificateVal|
	000001b0  69 64 69 74 79 50 65 72  69 6f 64 3a 20 38 37 36  |idityPeriod: 876|
	000001c0  30 68 30 6d 30 73 0a 63  65 72 74 69 66 69 63 61  |0h0m0s.certifica|
	000001d0  74 65 73 44 69 72 3a 20  2f 65 74 63 2f 6b 75 62  |tesDir: /etc/kub|
	000001e0  65 72 6e 65 74 65 73 2f  70 6b 69 0a 63 6c 75 73  |ernetes/pki.clus|
	000001f0  74 65 72 4e 61 6d 65 3a  20 6b 69 6e 64 0a 63 6f  |terName: kind.co|
	00000200  6e 74 72 6f 6c 50 6c 61  6e 65 45 6e 64 70 6f 69  |ntrolPlaneEndpoi|
	00000210  6e 74 3a 20 6b 69 6e 64  2d 63 6f 6e 74 72 6f 6c  |nt: kind-control|
	00000220  2d 70 6c 61 6e 65 3a 36  34 34 33 0a 63 6f 6e 74  |-plane:6443.cont|
	00000230  72 6f 6c 6c 65 72 4d 61  6e 61 67 65 72 3a 0a 20  |rollerManager:. |
	00000240  20 65 78 74 72 61 41 72  67 73 3a 0a 20 20 2d 20  | extraArgs:.  - |
	00000250  6e 61 6d 65 3a 20 65 6e  61 62 6c 65 2d 68 6f 73  |name: enable-hos|
	00000260  74 70 61 74 68 2d 70 72  6f 76 69 73 69 6f 6e 65  |tpath-provisione|
	00000270  72 0a 20 20 20 20 76 61  6c 75 65 3a 20 22 74 72  |r.    value: "tr|
	00000280  75 65 22 0a 64 6e 73 3a  20 7b 7d 0a 65 6e 63 72  |ue".dns: {}.encr|
	00000290  79 70 74 69 6f 6e 41 6c  67 6f 72 69 74 68 6d 3a  |yptionAlgorithm:|
	000002a0  20 52 53 41 2d 32 30 34  38 0a 65 74 63 64 3a 0a  | RSA-2048.etcd:.|
	000002b0  20 20 6c 6f 63 61 6c 3a  0a 20 20 20 20 64 61 74  |  local:.    dat|
	000002c0  61 44 69 72 3a 20 2f 76  61 72 2f 6c 69 62 2f 65  |aDir: /var/lib/e|
	000002d0  74 63 64 0a 69 6d 61 67  65 52 65 70 6f 73 69 74  |tcd.imageReposit|
	000002e0  6f 72 79 3a 20 72 65 67  69 73 74 72 79 2e 6b 38  |ory: registry.k8|
	000002f0  73 2e 69 6f 0a 6b 69 6e  64 3a 20 43 6c 75 73 74  |s.io.kind: Clust|
	00000300  65 72 43 6f 6e 66 69 67  75 72 61 74 69 6f 6e 0a  |erConfiguration.|
	00000310  6b 75 62 65 72 6e 65 74  65 73 56 65 72 73 69 6f  |kubernetesVersio|
	00000320  6e 3a 20 76 31 2e 33 34  2e 30 0a 6e 65 74 77 6f  |n: v1.34.0.netwo|
	00000330  72 6b 69 6e 67 3a 0a 20  20 64 6e 73 44 6f 6d 61  |rking:.  dnsDoma|
	00000340  69 6e 3a 20 63 6c 75 73  74 65 72 2e 6c 6f 63 61  |in: cluster.loca|
	00000350  6c 0a 20 20 70 6f 64 53  75 62 6e 65 74 3a 20 31  |l.  podSubnet: 1|
	00000360  30 2e 32 34 34 2e 30 2e  30 2f 31 36 0a 20 20 73  |0.244.0.0/16.  s|
	00000370  65 72 76 69 63 65 53 75  62 6e 65 74 3a 20 31 30  |erviceSubnet: 10|
	00000380  2e 39 36 2e 30 2e 30 2f  31 36 0a 70 72 6f 78 79  |.96.0.0/16.proxy|
	00000390  3a 20 7b 7d 0a 73 63 68  65 64 75 6c 65 72 3a 20  |: {}.scheduler: |
	000003a0  7b 7d 0a 1a 00 22 00                              |{}...".|
 >
I1215 14:50:15.313552     230 kubeproxy.go:55] attempting to download the KubeProxyConfiguration from ConfigMap "kube-proxy"
I1215 14:50:15.313605     230 type.go:165] "Request Body" body=""
I1215 14:50:15.313716     230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Authorization: Bearer <masked>" -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" 'https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kube-proxy?timeout=10s'
 >
I1215 14:50:15.315629     230 round_trippers.go:632] "Response" verb="GET" url="https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kube-proxy?timeout=10s" status="200 OK" headers=<
	Audit-Id: 7da1f078-b1a7-40b7-8b9d-595fb32485b5
	Cache-Control: no-cache, private
	Content-Length: 2089
	Content-Type: application/vnd.kubernetes.protobuf
	Date: Mon, 15 Dec 2025 14:50:15 GMT
	X-Kubernetes-Pf-Flowschema-Uid: f54c5d84-c2d8-4cbb-8801-07adb48ce73d
	X-Kubernetes-Pf-Prioritylevel-Uid: 7df0c0a7-2f8a-40ca-9625-0b6635feccd2
 > milliseconds=1 getConnectionMilliseconds=0 serverProcessingMilliseconds=1
I1215 14:50:15.315778     230 type.go:165] "Response Body" body=<
	00000000  6b 38 73 00 0a 0f 0a 02  76 31 12 09 43 6f 6e 66  |k8s.....v1..Conf|
	00000010  69 67 4d 61 70 12 8d 10  0a 85 02 0a 0a 6b 75 62  |igMap........kub|
	00000020  65 2d 70 72 6f 78 79 12  00 1a 0b 6b 75 62 65 2d  |e-proxy....kube-|
	00000030  73 79 73 74 65 6d 22 00  2a 24 34 31 37 66 31 65  |system".*$417f1e|
	00000040  38 32 2d 31 33 33 63 2d  34 65 38 32 2d 38 62 66  |82-133c-4e82-8bf|
	00000050  36 2d 38 66 34 36 34 31  33 31 33 61 66 37 32 03  |6-8f4641313af72.|
	00000060  32 33 38 38 00 42 08 08  d8 da ff c9 06 10 00 5a  |2388.B.........Z|
	00000070  11 0a 03 61 70 70 12 0a  6b 75 62 65 2d 70 72 6f  |...app..kube-pro|
	00000080  78 79 8a 01 9a 01 0a 07  6b 75 62 65 61 64 6d 12  |xy......kubeadm.|
	00000090  06 55 70 64 61 74 65 1a  02 76 31 22 08 08 d8 da  |.Update..v1"....|
	000000a0  ff c9 06 10 00 32 08 46  69 65 6c 64 73 56 31 3a  |.....2.FieldsV1:|
	000000b0  6d 0a 6b 7b 22 66 3a 64  61 74 61 22 3a 7b 22 2e  |m.k{"f:data":{".|
	000000c0  22 3a 7b 7d 2c 22 66 3a  63 6f 6e 66 69 67 2e 63  |":{},"f:config.c|
	000000d0  6f 6e 66 22 3a 7b 7d 2c  22 66 3a 6b 75 62 65 63  |onf":{},"f:kubec|
	000000e0  6f 6e 66 69 67 2e 63 6f  6e 66 22 3a 7b 7d 7d 2c  |onfig.conf":{}},|
	000000f0  22 66 3a 6d 65 74 61 64  61 74 61 22 3a 7b 22 66  |"f:metadata":{"f|
	00000100  3a 6c 61 62 65 6c 73 22  3a 7b 22 2e 22 3a 7b 7d  |:labels":{".":{}|
	00000110  2c 22 66 3a 61 70 70 22  3a 7b 7d 7d 7d 7d 42 00  |,"f:app":{}}}}B.|
	00000120  12 d1 0a 0a 0b 63 6f 6e  66 69 67 2e 63 6f 6e 66  |.....config.conf|
	00000130  12 c1 0a 61 70 69 56 65  72 73 69 6f 6e 3a 20 6b  |...apiVersion: k|
	00000140  75 62 65 70 72 6f 78 79  2e 63 6f 6e 66 69 67 2e  |ubeproxy.config.|
	00000150  6b 38 73 2e 69 6f 2f 76  31 61 6c 70 68 61 31 0a  |k8s.io/v1alpha1.|
	00000160  62 69 6e 64 41 64 64 72  65 73 73 3a 20 30 2e 30  |bindAddress: 0.0|
	00000170  2e 30 2e 30 0a 62 69 6e  64 41 64 64 72 65 73 73  |.0.0.bindAddress|
	00000180  48 61 72 64 46 61 69 6c  3a 20 66 61 6c 73 65 0a  |HardFail: false.|
	00000190  63 6c 69 65 6e 74 43 6f  6e 6e 65 63 74 69 6f 6e  |clientConnection|
	000001a0  3a 0a 20 20 61 63 63 65  70 74 43 6f 6e 74 65 6e  |:.  acceptConten|
	000001b0  74 54 79 70 65 73 3a 20  22 22 0a 20 20 62 75 72  |tTypes: "".  bur|
	000001c0  73 74 3a 20 30 0a 20 20  63 6f 6e 74 65 6e 74 54  |st: 0.  contentT|
	000001d0  79 70 65 3a 20 22 22 0a  20 20 6b 75 62 65 63 6f  |ype: "".  kubeco|
	000001e0  6e 66 69 67 3a 20 2f 76  61 72 2f 6c 69 62 2f 6b  |nfig: /var/lib/k|
	000001f0  75 62 65 2d 70 72 6f 78  79 2f 6b 75 62 65 63 6f  |ube-proxy/kubeco|
	00000200  6e 66 69 67 2e 63 6f 6e  66 0a 20 20 71 70 73 3a  |nfig.conf.  qps:|
	00000210  20 30 0a 63 6c 75 73 74  65 72 43 49 44 52 3a 20  | 0.clusterCIDR: |
	00000220  31 30 2e 32 34 34 2e 30  2e 30 2f 31 36 0a 63 6f  |10.244.0.0/16.co|
	00000230  6e 66 69 67 53 79 6e 63  50 65 72 69 6f 64 3a 20  |nfigSyncPeriod: |
	00000240  30 73 0a 63 6f 6e 6e 74  72 61 63 6b 3a 0a 20 20  |0s.conntrack:.  |
	00000250  6d 61 78 50 65 72 43 6f  72 65 3a 20 30 0a 20 20  |maxPerCore: 0.  |
	00000260  6d 69 6e 3a 20 6e 75 6c  6c 0a 20 20 74 63 70 42  |min: null.  tcpB|
	00000270  65 4c 69 62 65 72 61 6c  3a 20 66 61 6c 73 65 0a  |eLiberal: false.|
	00000280  20 20 74 63 70 43 6c 6f  73 65 57 61 69 74 54 69  |  tcpCloseWaitTi|
	00000290  6d 65 6f 75 74 3a 20 6e  75 6c 6c 0a 20 20 74 63  |meout: null.  tc|
	000002a0  70 45 73 74 61 62 6c 69  73 68 65 64 54 69 6d 65  |pEstablishedTime|
	000002b0  6f 75 74 3a 20 6e 75 6c  6c 0a 20 20 75 64 70 53  |out: null.  udpS|
	000002c0  74 72 65 61 6d 54 69 6d  65 6f 75 74 3a 20 30 73  |treamTimeout: 0s|
	000002d0  0a 20 20 75 64 70 54 69  6d 65 6f 75 74 3a 20 30  |.  udpTimeout: 0|
	000002e0  73 0a 64 65 74 65 63 74  4c 6f 63 61 6c 3a 0a 20  |s.detectLocal:. |
	000002f0  20 62 72 69 64 67 65 49  6e 74 65 72 66 61 63 65  | bridgeInterface|
	00000300  3a 20 22 22 0a 20 20 69  6e 74 65 72 66 61 63 65  |: "".  interface|
	00000310  4e 61 6d 65 50 72 65 66  69 78 3a 20 22 22 0a 64  |NamePrefix: "".d|
	00000320  65 74 65 63 74 4c 6f 63  61 6c 4d 6f 64 65 3a 20  |etectLocalMode: |
	00000330  22 22 0a 65 6e 61 62 6c  65 50 72 6f 66 69 6c 69  |"".enableProfili|
	00000340  6e 67 3a 20 66 61 6c 73  65 0a 68 65 61 6c 74 68  |ng: false.health|
	00000350  7a 42 69 6e 64 41 64 64  72 65 73 73 3a 20 22 22  |zBindAddress: ""|
	00000360  0a 68 6f 73 74 6e 61 6d  65 4f 76 65 72 72 69 64  |.hostnameOverrid|
	00000370  65 3a 20 22 22 0a 69 70  74 61 62 6c 65 73 3a 0a  |e: "".iptables:.|
	00000380  20 20 6c 6f 63 61 6c 68  6f 73 74 4e 6f 64 65 50  |  localhostNodeP|
	00000390  6f 72 74 73 3a 20 6e 75  6c 6c 0a 20 20 6d 61 73  |orts: null.  mas|
	000003a0  71 75 65 72 61 64 65 41  6c 6c 3a 20 66 61 6c 73  |queradeAll: fals|
	000003b0  65 0a 20 20 6d 61 73 71  75 65 72 61 64 65 42 69  |e.  masqueradeBi|
	000003c0  74 3a 20 6e 75 6c 6c 0a  20 20 6d 69 6e 53 79 6e  |t: null.  minSyn|
	000003d0  63 50 65 72 69 6f 64 3a  20 31 73 0a 20 20 73 79  |cPeriod: 1s.  sy|
	000003e0  6e 63 50 65 72 69 6f 64  3a 20 30 73 0a 69 70 76  |ncPeriod: 0s.ipv|
	000003f0  73 3a 0a 20 20 65 78 63  6c 75 64 65 43 49 44 52  |s:.  excludeCIDR|
	00000400  73 3a 20 6e 75 6c 6c 0a  20 20 6d 69 6e 53 79 6e  |s: null.  minSyn|
	00000410  63 50 65 72 69 6f 64 3a  20 30 73 0a 20 20 73 63  |cPeriod: 0s.  sc|
	00000420  68 65 64 75 6c 65 72 3a  20 22 22 0a 20 20 73 74  |heduler: "".  st|
	00000430  72 69 63 74 41 52 50 3a  20 66 61 6c 73 65 0a 20  |rictARP: false. |
	00000440  20 73 79 6e 63 50 65 72  69 6f 64 3a 20 30 73 0a  | syncPeriod: 0s.|
	00000450  20 20 74 63 70 46 69 6e  54 69 6d 65 6f 75 74 3a  |  tcpFinTimeout:|
	00000460  20 30 73 0a 20 20 74 63  70 54 69 6d 65 6f 75 74  | 0s.  tcpTimeout|
	00000470  3a 20 30 73 0a 20 20 75  64 70 54 69 6d 65 6f 75  |: 0s.  udpTimeou|
	00000480  74 3a 20 30 73 0a 6b 69  6e 64 3a 20 4b 75 62 65  |t: 0s.kind: Kube|
	00000490  50 72 6f 78 79 43 6f 6e  66 69 67 75 72 61 74 69  |ProxyConfigurati|
	000004a0  6f 6e 0a 6c 6f 67 67 69  6e 67 3a 0a 20 20 66 6c  |on.logging:.  fl|
	000004b0  75 73 68 46 72 65 71 75  65 6e 63 79 3a 20 30 0a  |ushFrequency: 0.|
	000004c0  20 20 6f 70 74 69 6f 6e  73 3a 0a 20 20 20 20 6a  |  options:.    j|
	000004d0  73 6f 6e 3a 0a 20 20 20  20 20 20 69 6e 66 6f 42  |son:.      infoB|
	000004e0  75 66 66 65 72 53 69 7a  65 3a 20 22 30 22 0a 20  |ufferSize: "0". |
	000004f0  20 20 20 74 65 78 74 3a  0a 20 20 20 20 20 20 69  |   text:.      i|
	00000500  6e 66 6f 42 75 66 66 65  72 53 69 7a 65 3a 20 22  |nfoBufferSize: "|
	00000510  30 22 0a 20 20 76 65 72  62 6f 73 69 74 79 3a 20  |0".  verbosity: |
	00000520  30 0a 6d 65 74 72 69 63  73 42 69 6e 64 41 64 64  |0.metricsBindAdd|
	00000530  72 65 73 73 3a 20 22 22  0a 6d 6f 64 65 3a 20 69  |ress: "".mode: i|
	00000540  70 74 61 62 6c 65 73 0a  6e 66 74 61 62 6c 65 73  |ptables.nftables|
	00000550  3a 0a 20 20 6d 61 73 71  75 65 72 61 64 65 41 6c  |:.  masqueradeAl|
	00000560  6c 3a 20 66 61 6c 73 65  0a 20 20 6d 61 73 71 75  |l: false.  masqu|
	00000570  65 72 61 64 65 42 69 74  3a 20 6e 75 6c 6c 0a 20  |eradeBit: null. |
	00000580  20 6d 69 6e 53 79 6e 63  50 65 72 69 6f 64 3a 20  | minSyncPeriod: |
	00000590  30 73 0a 20 20 73 79 6e  63 50 65 72 69 6f 64 3a  |0s.  syncPeriod:|
	000005a0  20 30 73 0a 6e 6f 64 65  50 6f 72 74 41 64 64 72  | 0s.nodePortAddr|
	000005b0  65 73 73 65 73 3a 20 6e  75 6c 6c 0a 6f 6f 6d 53  |esses: null.oomS|
	000005c0  63 6f 72 65 41 64 6a 3a  20 6e 75 6c 6c 0a 70 6f  |coreAdj: null.po|
	000005d0  72 74 52 61 6e 67 65 3a  20 22 22 0a 73 68 6f 77  |rtRange: "".show|
	000005e0  48 69 64 64 65 6e 4d 65  74 72 69 63 73 46 6f 72  |HiddenMetricsFor|
	000005f0  56 65 72 73 69 6f 6e 3a  20 22 22 0a 77 69 6e 6b  |Version: "".wink|
	00000600  65 72 6e 65 6c 3a 0a 20  20 65 6e 61 62 6c 65 44  |ernel:.  enableD|
	00000610  53 52 3a 20 66 61 6c 73  65 0a 20 20 66 6f 72 77  |SR: false.  forw|
	00000620  61 72 64 48 65 61 6c 74  68 43 68 65 63 6b 56 69  |ardHealthCheckVi|
	00000630  70 3a 20 66 61 6c 73 65  0a 20 20 6e 65 74 77 6f  |p: false.  netwo|
	00000640  72 6b 4e 61 6d 65 3a 20  22 22 0a 20 20 72 6f 6f  |rkName: "".  roo|
	00000650  74 48 6e 73 45 6e 64 70  6f 69 6e 74 4e 61 6d 65  |tHnsEndpointName|
	00000660  3a 20 22 22 0a 20 20 73  6f 75 72 63 65 56 69 70  |: "".  sourceVip|
	00000670  3a 20 22 22 12 ae 03 0a  0f 6b 75 62 65 63 6f 6e  |: "".....kubecon|
	00000680  66 69 67 2e 63 6f 6e 66  12 9a 03 61 70 69 56 65  |fig.conf...apiVe|
	00000690  72 73 69 6f 6e 3a 20 76  31 0a 6b 69 6e 64 3a 20  |rsion: v1.kind: |
	000006a0  43 6f 6e 66 69 67 0a 63  6c 75 73 74 65 72 73 3a  |Config.clusters:|
	000006b0  0a 2d 20 63 6c 75 73 74  65 72 3a 0a 20 20 20 20  |.- cluster:.    |
	000006c0  63 65 72 74 69 66 69 63  61 74 65 2d 61 75 74 68  |certificate-auth|
	000006d0  6f 72 69 74 79 3a 20 2f  76 61 72 2f 72 75 6e 2f  |ority: /var/run/|
	000006e0  73 65 63 72 65 74 73 2f  6b 75 62 65 72 6e 65 74  |secrets/kubernet|
	000006f0  65 73 2e 69 6f 2f 73 65  72 76 69 63 65 61 63 63  |es.io/serviceacc|
	00000700  6f 75 6e 74 2f 63 61 2e  63 72 74 0a 20 20 20 20  |ount/ca.crt.    |
	00000710  73 65 72 76 65 72 3a 20  68 74 74 70 73 3a 2f 2f  |server: https://|
	00000720  6b 69 6e 64 2d 63 6f 6e  74 72 6f 6c 2d 70 6c 61  |kind-control-pla|
	00000730  6e 65 3a 36 34 34 33 0a  20 20 6e 61 6d 65 3a 20  |ne:6443.  name: |
	00000740  64 65 66 61 75 6c 74 0a  63 6f 6e 74 65 78 74 73  |default.contexts|
	00000750  3a 0a 2d 20 63 6f 6e 74  65 78 74 3a 0a 20 20 20  |:.- context:.   |
	00000760  20 63 6c 75 73 74 65 72  3a 20 64 65 66 61 75 6c  | cluster: defaul|
	00000770  74 0a 20 20 20 20 6e 61  6d 65 73 70 61 63 65 3a  |t.    namespace:|
	00000780  20 64 65 66 61 75 6c 74  0a 20 20 20 20 75 73 65  | default.    use|
	00000790  72 3a 20 64 65 66 61 75  6c 74 0a 20 20 6e 61 6d  |r: default.  nam|
	000007a0  65 3a 20 64 65 66 61 75  6c 74 0a 63 75 72 72 65  |e: default.curre|
	000007b0  6e 74 2d 63 6f 6e 74 65  78 74 3a 20 64 65 66 61  |nt-context: defa|
	000007c0  75 6c 74 0a 75 73 65 72  73 3a 0a 2d 20 6e 61 6d  |ult.users:.- nam|
	000007d0  65 3a 20 64 65 66 61 75  6c 74 0a 20 20 75 73 65  |e: default.  use|
	000007e0  72 3a 0a 20 20 20 20 74  6f 6b 65 6e 46 69 6c 65  |r:.    tokenFile|
	000007f0  3a 20 2f 76 61 72 2f 72  75 6e 2f 73 65 63 72 65  |: /var/run/secre|
	00000800  74 73 2f 6b 75 62 65 72  6e 65 74 65 73 2e 69 6f  |ts/kubernetes.io|
	00000810  2f 73 65 72 76 69 63 65  61 63 63 6f 75 [truncated 102 chars]
 >
I1215 14:50:15.319974     230 kubelet.go:73] attempting to download the KubeletConfiguration from ConfigMap "kubelet-config"
I1215 14:50:15.320052     230 type.go:165] "Request Body" body=""
I1215 14:50:15.320143     230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" -H "Authorization: Bearer <masked>" 'https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubelet-config?timeout=10s'
 >
I1215 14:50:15.322275     230 round_trippers.go:632] "Response" verb="GET" url="https://kind-control-plane:6443/api/v1/namespaces/kube-system/configmaps/kubelet-config?timeout=10s" status="200 OK" headers=<
	Audit-Id: edacaa09-2a92-49f8-992d-467253d02bac
	Cache-Control: no-cache, private
	Content-Length: 1453
	Content-Type: application/vnd.kubernetes.protobuf
	Date: Mon, 15 Dec 2025 14:50:15 GMT
	X-Kubernetes-Pf-Flowschema-Uid: f54c5d84-c2d8-4cbb-8801-07adb48ce73d
	X-Kubernetes-Pf-Prioritylevel-Uid: 7df0c0a7-2f8a-40ca-9625-0b6635feccd2
 > milliseconds=2 getConnectionMilliseconds=0 serverProcessingMilliseconds=1
I1215 14:50:15.322454     230 type.go:165] "Response Body" body=<
	00000000  6b 38 73 00 0a 0f 0a 02  76 31 12 09 43 6f 6e 66  |k8s.....v1..Conf|
	00000010  69 67 4d 61 70 12 91 0b  0a ac 01 0a 0e 6b 75 62  |igMap........kub|
	00000020  65 6c 65 74 2d 63 6f 6e  66 69 67 12 00 1a 0b 6b  |elet-config....k|
	00000030  75 62 65 2d 73 79 73 74  65 6d 22 00 2a 24 30 32  |ube-system".*$02|
	00000040  36 61 32 32 32 63 2d 39  66 65 34 2d 34 35 39 37  |6a222c-9fe4-4597|
	00000050  2d 39 30 38 61 2d 62 35  65 35 64 62 33 35 32 35  |-908a-b5e5db3525|
	00000060  36 64 32 03 32 31 30 38  00 42 08 08 d7 da ff c9  |6d2.2108.B......|
	00000070  06 10 00 8a 01 51 0a 07  6b 75 62 65 61 64 6d 12  |.....Q..kubeadm.|
	00000080  06 55 70 64 61 74 65 1a  02 76 31 22 08 08 d7 da  |.Update..v1"....|
	00000090  ff c9 06 10 00 32 08 46  69 65 6c 64 73 56 31 3a  |.....2.FieldsV1:|
	000000a0  24 0a 22 7b 22 66 3a 64  61 74 61 22 3a 7b 22 2e  |$."{"f:data":{".|
	000000b0  22 3a 7b 7d 2c 22 66 3a  6b 75 62 65 6c 65 74 22  |":{},"f:kubelet"|
	000000c0  3a 7b 7d 7d 7d 42 00 12  df 09 0a 07 6b 75 62 65  |:{}}}B......kube|
	000000d0  6c 65 74 12 d3 09 61 70  69 56 65 72 73 69 6f 6e  |let...apiVersion|
	000000e0  3a 20 6b 75 62 65 6c 65  74 2e 63 6f 6e 66 69 67  |: kubelet.config|
	000000f0  2e 6b 38 73 2e 69 6f 2f  76 31 62 65 74 61 31 0a  |.k8s.io/v1beta1.|
	00000100  61 75 74 68 65 6e 74 69  63 61 74 69 6f 6e 3a 0a  |authentication:.|
	00000110  20 20 61 6e 6f 6e 79 6d  6f 75 73 3a 0a 20 20 20  |  anonymous:.   |
	00000120  20 65 6e 61 62 6c 65 64  3a 20 66 61 6c 73 65 0a  | enabled: false.|
	00000130  20 20 77 65 62 68 6f 6f  6b 3a 0a 20 20 20 20 63  |  webhook:.    c|
	00000140  61 63 68 65 54 54 4c 3a  20 30 73 0a 20 20 20 20  |acheTTL: 0s.    |
	00000150  65 6e 61 62 6c 65 64 3a  20 74 72 75 65 0a 20 20  |enabled: true.  |
	00000160  78 35 30 39 3a 0a 20 20  20 20 63 6c 69 65 6e 74  |x509:.    client|
	00000170  43 41 46 69 6c 65 3a 20  2f 65 74 63 2f 6b 75 62  |CAFile: /etc/kub|
	00000180  65 72 6e 65 74 65 73 2f  70 6b 69 2f 63 61 2e 63  |ernetes/pki/ca.c|
	00000190  72 74 0a 61 75 74 68 6f  72 69 7a 61 74 69 6f 6e  |rt.authorization|
	000001a0  3a 0a 20 20 6d 6f 64 65  3a 20 57 65 62 68 6f 6f  |:.  mode: Webhoo|
	000001b0  6b 0a 20 20 77 65 62 68  6f 6f 6b 3a 0a 20 20 20  |k.  webhook:.   |
	000001c0  20 63 61 63 68 65 41 75  74 68 6f 72 69 7a 65 64  | cacheAuthorized|
	000001d0  54 54 4c 3a 20 30 73 0a  20 20 20 20 63 61 63 68  |TTL: 0s.    cach|
	000001e0  65 55 6e 61 75 74 68 6f  72 69 7a 65 64 54 54 4c  |eUnauthorizedTTL|
	000001f0  3a 20 30 73 0a 63 67 72  6f 75 70 44 72 69 76 65  |: 0s.cgroupDrive|
	00000200  72 3a 20 73 79 73 74 65  6d 64 0a 63 67 72 6f 75  |r: systemd.cgrou|
	00000210  70 52 6f 6f 74 3a 20 2f  6b 75 62 65 6c 65 74 0a  |pRoot: /kubelet.|
	00000220  63 6c 75 73 74 65 72 44  4e 53 3a 0a 2d 20 31 30  |clusterDNS:.- 10|
	00000230  2e 39 36 2e 30 2e 31 30  0a 63 6c 75 73 74 65 72  |.96.0.10.cluster|
	00000240  44 6f 6d 61 69 6e 3a 20  63 6c 75 73 74 65 72 2e  |Domain: cluster.|
	00000250  6c 6f 63 61 6c 0a 63 6f  6e 74 61 69 6e 65 72 52  |local.containerR|
	00000260  75 6e 74 69 6d 65 45 6e  64 70 6f 69 6e 74 3a 20  |untimeEndpoint: |
	00000270  22 22 0a 63 70 75 4d 61  6e 61 67 65 72 52 65 63  |"".cpuManagerRec|
	00000280  6f 6e 63 69 6c 65 50 65  72 69 6f 64 3a 20 30 73  |oncilePeriod: 0s|
	00000290  0a 63 72 61 73 68 4c 6f  6f 70 42 61 63 6b 4f 66  |.crashLoopBackOf|
	000002a0  66 3a 20 7b 7d 0a 65 76  69 63 74 69 6f 6e 48 61  |f: {}.evictionHa|
	000002b0  72 64 3a 0a 20 20 69 6d  61 67 65 66 73 2e 61 76  |rd:.  imagefs.av|
	000002c0  61 69 6c 61 62 6c 65 3a  20 30 25 0a 20 20 6e 6f  |ailable: 0%.  no|
	000002d0  64 65 66 73 2e 61 76 61  69 6c 61 62 6c 65 3a 20  |defs.available: |
	000002e0  30 25 0a 20 20 6e 6f 64  65 66 73 2e 69 6e 6f 64  |0%.  nodefs.inod|
	000002f0  65 73 46 72 65 65 3a 20  30 25 0a 65 76 69 63 74  |esFree: 0%.evict|
	00000300  69 6f 6e 50 72 65 73 73  75 72 65 54 72 61 6e 73  |ionPressureTrans|
	00000310  69 74 69 6f 6e 50 65 72  69 6f 64 3a 20 30 73 0a  |itionPeriod: 0s.|
	00000320  66 61 69 6c 53 77 61 70  4f 6e 3a 20 66 61 6c 73  |failSwapOn: fals|
	00000330  65 0a 66 69 6c 65 43 68  65 63 6b 46 72 65 71 75  |e.fileCheckFrequ|
	00000340  65 6e 63 79 3a 20 30 73  0a 68 65 61 6c 74 68 7a  |ency: 0s.healthz|
	00000350  42 69 6e 64 41 64 64 72  65 73 73 3a 20 31 32 37  |BindAddress: 127|
	00000360  2e 30 2e 30 2e 31 0a 68  65 61 6c 74 68 7a 50 6f  |.0.0.1.healthzPo|
	00000370  72 74 3a 20 31 30 32 34  38 0a 68 74 74 70 43 68  |rt: 10248.httpCh|
	00000380  65 63 6b 46 72 65 71 75  65 6e 63 79 3a 20 30 73  |eckFrequency: 0s|
	00000390  0a 69 6d 61 67 65 47 43  48 69 67 68 54 68 72 65  |.imageGCHighThre|
	000003a0  73 68 6f 6c 64 50 65 72  63 65 6e 74 3a 20 31 30  |sholdPercent: 10|
	000003b0  30 0a 69 6d 61 67 65 4d  61 78 69 6d 75 6d 47 43  |0.imageMaximumGC|
	000003c0  41 67 65 3a 20 30 73 0a  69 6d 61 67 65 4d 69 6e  |Age: 0s.imageMin|
	000003d0  69 6d 75 6d 47 43 41 67  65 3a 20 30 73 0a 6b 69  |imumGCAge: 0s.ki|
	000003e0  6e 64 3a 20 4b 75 62 65  6c 65 74 43 6f 6e 66 69  |nd: KubeletConfi|
	000003f0  67 75 72 61 74 69 6f 6e  0a 6c 6f 67 67 69 6e 67  |guration.logging|
	00000400  3a 0a 20 20 66 6c 75 73  68 46 72 65 71 75 65 6e  |:.  flushFrequen|
	00000410  63 79 3a 20 30 0a 20 20  6f 70 74 69 6f 6e 73 3a  |cy: 0.  options:|
	00000420  0a 20 20 20 20 6a 73 6f  6e 3a 0a 20 20 20 20 20  |.    json:.     |
	00000430  20 69 6e 66 6f 42 75 66  66 65 72 53 69 7a 65 3a  | infoBufferSize:|
	00000440  20 22 30 22 0a 20 20 20  20 74 65 78 74 3a 0a 20  | "0".    text:. |
	00000450  20 20 20 20 20 69 6e 66  6f 42 75 66 66 65 72 53  |     infoBufferS|
	00000460  69 7a 65 3a 20 22 30 22  0a 20 20 76 65 72 62 6f  |ize: "0".  verbo|
	00000470  73 69 74 79 3a 20 30 0a  6d 65 6d 6f 72 79 53 77  |sity: 0.memorySw|
	00000480  61 70 3a 20 7b 7d 0a 6e  6f 64 65 53 74 61 74 75  |ap: {}.nodeStatu|
	00000490  73 52 65 70 6f 72 74 46  72 65 71 75 65 6e 63 79  |sReportFrequency|
	000004a0  3a 20 30 73 0a 6e 6f 64  65 53 74 61 74 75 73 55  |: 0s.nodeStatusU|
	000004b0  70 64 61 74 65 46 72 65  71 75 65 6e 63 79 3a 20  |pdateFrequency: |
	000004c0  30 73 0a 72 6f 74 61 74  65 43 65 72 74 69 66 69  |0s.rotateCertifi|
	000004d0  63 61 74 65 73 3a 20 74  72 75 65 0a 72 75 6e 74  |cates: true.runt|
	000004e0  69 6d 65 52 65 71 75 65  73 74 54 69 6d 65 6f 75  |imeRequestTimeou|
	000004f0  74 3a 20 30 73 0a 73 68  75 74 64 6f 77 6e 47 72  |t: 0s.shutdownGr|
	00000500  61 63 65 50 65 72 69 6f  64 3a 20 30 73 0a 73 68  |acePeriod: 0s.sh|
	00000510  75 74 64 6f 77 6e 47 72  61 63 65 50 65 72 69 6f  |utdownGracePerio|
	00000520  64 43 72 69 74 69 63 61  6c 50 6f 64 73 3a 20 30  |dCriticalPods: 0|
	00000530  73 0a 73 74 61 74 69 63  50 6f 64 50 61 74 68 3a  |s.staticPodPath:|
	00000540  20 2f 65 74 63 2f 6b 75  62 65 72 6e 65 74 65 73  | /etc/kubernetes|
	00000550  2f 6d 61 6e 69 66 65 73  74 73 0a 73 74 72 65 61  |/manifests.strea|
	00000560  6d 69 6e 67 43 6f 6e 6e  65 63 74 69 6f 6e 49 64  |mingConnectionId|
	00000570  6c 65 54 69 6d 65 6f 75  74 3a 20 30 73 0a 73 79  |leTimeout: 0s.sy|
	00000580  6e 63 46 72 65 71 75 65  6e 63 79 3a 20 30 73 0a  |ncFrequency: 0s.|
	00000590  76 6f 6c 75 6d 65 53 74  61 74 73 41 67 67 50 65  |volumeStatsAggPe|
	000005a0  72 69 6f 64 3a 20 30 73  0a 1a 00 22 00           |riod: 0s...".|
 >
I1215 14:50:15.324583     230 initconfiguration.go:114] skip CRI socket detection, fill with the default CRI socket unix:///var/run/containerd/containerd.sock
I1215 14:50:15.324815     230 interface.go:432] Looking for default routes with IPv4 addresses
I1215 14:50:15.324830     230 interface.go:437] Default route transits interface "eth0"
I1215 14:50:15.324975     230 interface.go:209] Interface eth0 is up
I1215 14:50:15.325021     230 interface.go:257] Interface "eth0" has 3 addresses :[172.18.0.3/16 fc00:f853:ccd:e793::3/64 fe80::5092:11ff:fe0b:4722/64].
I1215 14:50:15.325041     230 interface.go:224] Checking addr  172.18.0.3/16.
I1215 14:50:15.325057     230 interface.go:231] IP found 172.18.0.3
I1215 14:50:15.325072     230 interface.go:263] Found valid IPv4 address 172.18.0.3 for interface "eth0".
I1215 14:50:15.325085     230 interface.go:443] Found active IP 172.18.0.3 
I1215 14:50:15.325133     230 common.go:148] WARNING: tolerating control plane version v1.34.0 as a pre-release version
I1215 14:50:15.325168     230 preflight.go:108] [preflight] Running configuration dependant checks
I1215 14:50:15.325179     230 controlplaneprepare.go:225] [download-certs] Skipping certs download
I1215 14:50:15.325485     230 kubelet.go:147] [kubelet-start] writing bootstrap kubelet config file at /etc/kubernetes/bootstrap-kubelet.conf
I1215 14:50:15.327017     230 kubelet.go:162] [kubelet-start] writing CA certificate at /etc/kubernetes/pki/ca.crt
I1215 14:50:15.327125     230 kubelet.go:178] [kubelet-start] Checking for an existing Node in the cluster with name "5a5adbbfec7a" and status "Ready"
I1215 14:50:15.327178     230 type.go:165] "Request Body" body=""
I1215 14:50:15.327263     230 round_trippers.go:527] "Request" curlCommand=<
	curl -v -XGET  -H "Accept: application/vnd.kubernetes.protobuf,application/json" -H "User-Agent: kubeadm/v1.35.0 (linux/amd64) kubernetes/f35f950" -H "Authorization: Bearer <masked>" 'https://kind-control-plane:6443/api/v1/nodes/5a5adbbfec7a?timeout=10s'
 >
I1215 14:50:15.329075     230 round_trippers.go:632] "Response" verb="GET" url="https://kind-control-plane:6443/api/v1/nodes/5a5adbbfec7a?timeout=10s" status="404 Not Found" headers=<
	Audit-Id: b5dddad2-5b97-4e03-9241-52b266d62df5
	Cache-Control: no-cache, private
	Content-Length: 115
	Content-Type: application/vnd.kubernetes.protobuf
	Date: Mon, 15 Dec 2025 14:50:15 GMT
	X-Kubernetes-Pf-Flowschema-Uid: f54c5d84-c2d8-4cbb-8801-07adb48ce73d
	X-Kubernetes-Pf-Prioritylevel-Uid: 7df0c0a7-2f8a-40ca-9625-0b6635feccd2
 > milliseconds=1 getConnectionMilliseconds=0 serverProcessingMilliseconds=1
I1215 14:50:15.329149     230 type.go:165] "Response Body" body=<
	00000000  6b 38 73 00 0a 0c 0a 02  76 31 12 06 53 74 61 74  |k8s.....v1..Stat|
	00000010  75 73 12 5b 0a 06 0a 00  12 00 1a 00 12 07 46 61  |us.[..........Fa|
	00000020  69 6c 75 72 65 1a 1e 6e  6f 64 65 73 20 22 35 61  |ilure..nodes "5a|
	00000030  35 61 64 62 62 66 65 63  37 61 22 20 6e 6f 74 20  |5adbbfec7a" not |
	00000040  66 6f 75 6e 64 22 08 4e  6f 74 46 6f 75 6e 64 2a  |found".NotFound*|
	00000050  1b 0a 0c 35 61 35 61 64  62 62 66 65 63 37 61 12  |...5a5adbbfec7a.|
	00000060  00 1a 05 6e 6f 64 65 73  28 00 32 00 30 94 03 1a  |...nodes(.2.0...|
	00000070  00 22 00                                          |.".|
 >
I1215 14:50:15.329251     230 kubelet.go:193] [kubelet-start] Stopping the kubelet
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 501.664771ms
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap
I1215 14:50:15.990092     230 loader.go:405] Config loaded from file:  /etc/kubernetes/kubelet.conf
I1215 14:50:15.990751     230 cert_rotation.go:141] "Starting client certificate rotation controller" logger="tls-transport-cache"
I1215 14:50:15.991227     230 loader.go:405] Config loaded from file:  /etc/kubernetes/kubelet.conf

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
{% endraw %}
</details>
