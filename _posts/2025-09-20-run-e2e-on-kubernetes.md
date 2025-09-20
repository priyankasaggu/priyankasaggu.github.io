---
layout: post
title: "How to run Kubernetes (Node) e2e tests locally?"
tags: [kubernetes]
comments: false
---

***TL;DR:***

- To run Node e2e tests

  ***without feature-gate***

  ```bash
  > KUBE_ROOT=$(pwd) make test-e2e-node FOCUS="a pod failing to mount volumes"

  ```
 
  ***with feature-gate***

  ```bash
  > KUBE_ROOT=$(pwd) make test-e2e-node FOCUS="a pod failing to mount volumes" TEST_ARGS='--feature-gates=PodReadyToStartContainersCondition=true'
  ```


- To run rest of the e2e tests:

  ```bash
  > ./_output/bin/e2e.test \
  --provider=local \
  --kubeconfig=$HOME/.kube/config \
  --ginkgo.focus="PodReadyToStartContainerCondition"
  ```

---
---

(logging for my future self!)

Below are a bunch of my raw running notes as I try to figure out how to run Kubernetes e2e tests locally.

I'm trying to add a new e2e test in the Kubernetes codebase (as part of the work required for the GA promotion of KEP-3085 (aka the feature-gate, `PodReadyToStartContainerCondition`).  

And I want to be able to run this locally.

So, starting with the following steps to set up my environment:

- clone the kubernetes repository

  ```bash
  > git clone git@github.com:kubernetes/kubernetes.git

  > cd kubernetes

  > git checkout -b some-new-branch-name

  # build all the binaries you'll need for running e2e locally
  > make WHAT=test/e2e/e2e.test

  ```

  This will produce the `e2e.test` binary in `./output/bin/e2e.test` path.


- Set up a local kubernetes cluster (I'll use `kind`)

  ```bash
  > kind create cluster --name local-e2e-test

  # to make sure my kubeconfig is pointing to the intended cluster
  > kubectl cluster-info
  ```

- Run the e2e test binary (the newly built `./output/bin/e2e.test` binary).

  The e2e framework runs Ginkgo tests.

  I can target the test I want to run, by its `--ginkgo.focus` flag.

  Example:

  ```bash
  > ./_output/bin/e2e.test \
  --provider=local \
  --kubeconfig=$HOME/.kube/config \
  --ginkgo.focus="PodReadyToStartContainerCondition"
  ```

  I also learnt, that there's a `--help` flag also available, if I want to understand the full length of the capablities offered by this `e2e.test` binary.

  I tried a few of them below.

  For example, If I want to get a dump of the entire list of e2e tests from the binary:

  ```bash
  > ./_output/bin/e2e.test --list-tests
  ```

  That prints all 7k+ test names.

  So, if I want to check for a specific test, I can grep for some keyword or string used in the test definition (I learnt this needs to some keyword or string from the `ginkgo.It(...)` block where the test is invoked):

  ```bash
  > ./_output/bin/e2e.test --list-tests | grep -i "some-string-in-the-test-definition"
  ```

  Also, ***please note***, if I have added a new test after above steps, then IIUC `e2e.test --list-tests` won't show it right away.  
  I'll need to rebuild the binary to make the new test visible.


---

You know what? I realised what I did above, doesn't really help me to run the Node e2e tests (aka the tests defined in the path `test/e2d_node/*` of the Kubernetes repository).

(And for the KEP-3085, that is exactly what I need - to be able to write and run Node e2e tests locally)

But before I move ahead - How did I realise that I need something else?

Well, because - `e2e.test` binary wasn't able to list my intended test (which is defined in the `test/e2e_node/pod_conditions_test.go` path in the Kubernetes repo).

Ok, moving forward.

I figured out, there's a different way to run these Node e2e tests locally.

I came across this handy document – [Node End-To-End (e2e) tests](https://github.com/kubernetes/community/blob/d71ff1332c77beebcdbeaa2b7004453a2a136d1f/contributors/devel/sig-node/e2e-node-tests.md?plain=1#L1).  
This very thoroughly describes the process for running Node e2e tests locally (as well as, remotely, but that's for some other day. I want to try the local execution first).
(Thank you, to the good folks in the SIG Node, this truly is a gem of a resource!)

Ok, let's see if this helps me with my usecase!

There's a bit of work needed (the pre-requisities) before I can start following the instructions in the document to actually run the tests.

- I need to install etcd, and make sure it's available in the path.  
  (I'm on an openSUSE Tumbleweed machine, so I did `zypper install etcd` and that did the job. Otherwise, follow the instructions from Etcd documentation [here](https://etcd.io/docs/v3.6/install/).)

  ```bash
  > which etcd
  /usr/sbin/etcd
  
  > etcd --version
  etcd Version: 3.6.4
  Git SHA: Not provided (use ./build instead of go build)
  Go Version: go1.24.5
  Go OS/Arch: linux/amd64
  ```
- Next, I need to install Containerd and configure cgroup driver to cgroupfs.  
  (once again, I did `zypper install containerd`. Otherwise, follow the instructions from Containerd documentation [here](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).)

  ```bash
  > sudo mkdir -p /etc/containerd
  > sudo containerd config default | sudo tee /etc/containerd/config.toml
  > sudo sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml
  > sudo systemctl restart containerd

  > containerd --version
  containerd github.com/containerd/containerd v1.7.27 05044ec0a9a75232cad458027ca83437aae3f4da
  ```

- And then finally, I need to ensure that there's a valid CNI configuration in `/etc/cni/net.d/`.  
  For that I ran the following steps.

  ```bash
  > sudo mkdir -p /opt/cni/bin

  > sudo curl -L https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz | sudo tar -C /opt/cni/bin -xz

  > sudo mkdir -p /etc/cni/net.d

  > sudo tee /etc/cni/net.d/10-bridge.conf <<EOF
  {
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [{"dst": "0.0.0.0/0"}]
    }
  }
  EOF
  ```


With these pre-requisties out of my way, for running the Node e2e tests (i.e., to run the *ginkgo* binary against the subdirectory *test/e2e_node*), I will use the following `make` target:

```bash
make test-e2e-node
```

And this also has a handy `HELP` option:

```bash
make test-e2e-node PRINT_HELP=y
```

Right now, I want to try running a single test, so, I will use the `FOCUS` arugment to target it.  
(I want to test [this block](https://github.com/kubernetes/kubernetes/blob/09aaf7226056a7964adcb176d789de5507313d00/test/e2e_node/pod_conditions_test.go#L140-L199) invoked via this `ginkgo.It(...)` [description](https://github.com/kubernetes/kubernetes/blob/09aaf7226056a7964adcb176d789de5507313d00/test/e2e_node/pod_conditions_test.go#L61-L69))

So, my make target call would look something like:  
(Please note, I also had to set the `KUBE_ROOT` variable to point to the correct path to my local kubernetes repo clone.)


```bash
KUBE_ROOT=$(pwd) make test-e2e-node FOCUS="just the scheduled condition set"
```

Wait! It ran, but not successfully. I got the following error:

```bash
  << Captured StdOut/StdErr Output

  Timeline >>
  I0920 14:50:41.511206   22650 server.go:105] Starting server "services" with command "/home/psaggu/work-upstream/kep-3085-add-condition-for-sandbox-creation/kubernetes/_output/local/go/bin/e2e_node.test --run-services-mode --bearer-token=Ai-zpfsimz4_NvH2 --test.timeout=0 --ginkgo.seed=1758360041 --ginkgo.timeout=23h59m59.999947072s --ginkgo.grace-period=30s --ginkgo.focus=just the scheduled condition set --ginkgo.skip=\\[Flaky\\]|\\[Slow\\]|\\[Serial\\] --ginkgo.parallel.process=1 --ginkgo.parallel.total=8 --ginkgo.parallel.host=127.0.0.1:33109 --v 4 --report-dir=/tmp/_artifacts/250920T145006 --node-name alpha --kubelet-flags=--cluster-domain=cluster.local --dns-domain=cluster.local --prepull-images=false --container-runtime-endpoint=unix:///run/containerd/containerd.sock --runtime-config= --kubelet-config-file=test/e2e_node/jenkins/default-kubelet-config.yaml"
  I0920 14:50:41.511254   22650 util.go:48] Running readiness check for service "services"
  I0920 14:50:41.511354   22650 server.go:133] Output file for server "services": /tmp/_artifacts/250920T145006/services.log
  I0920 14:50:41.511706   22650 server.go:163] Waiting for server "services" start command to complete
  W0920 14:50:43.029782   22650 util.go:106] Health check on "https://127.0.0.1:6443/healthz" failed, status=500
  I0920 14:50:44.032240   22650 services.go:69] Node services started.
  I0920 14:50:44.032254   22650 kubelet.go:157] Starting kubelet
  I0920 14:50:44.032259 22650 kubelet.go:159] Standalone mode: false
  I0920 14:50:44.033031   22650 feature_gate.go:385] feature gates: {map[]}
  I0920 14:50:44.042420   22650 server.go:105] Starting server "kubelet" with command "/usr/bin/systemd-run -p Delegate=true -p StandardError=append:/tmp/_artifacts/250920T145006/kubelet.log --unit=kubelet-20250920T145044.service --slice=runtime.slice --remain-after-exit /home/psaggu/work-upstream/kep-3085-add-condition-for-sandbox-creation/kubernetes/_output/local/go/bin/kubelet --kubeconfig /home/psaggu/work-upstream/kep-3085-add-condition-for-sandbox-creation/kubernetes/_output/local/go/bin/kubeconfig --root-dir /var/lib/kubelet --v 4 --config-dir /home/psaggu/work-upstream/kep-3085-add-condition-for-sandbox-creation/kubernetes/_output/local/go/bin/kubelet.conf.d --hostname-override alpha --container-runtime-endpoint unix:///run/containerd/containerd.sock --config /home/psaggu/work-upstream/kep-3085-add-condition-for-sandbox-creation/kubernetes/_output/local/go/bin/kubelet-config --cluster-domain=cluster.local"
  I0920 14:50:44.042464   22650 util.go:48] Running readiness check for service "kubelet"
  I0920 14:50:44.042571   22650 server.go:133] Output file for server "kubelet": /tmp/_artifacts/250920T145006/kubelet.log
  I0920 14:50:44.043021   22650 server.go:163] Waiting for server "kubelet" start command to complete
  W0920 14:50:45.058069   22650 util.go:104] Health check on "http://127.0.0.1:10248/healthz" failed, error=Head "http://127.0.0.1:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused

    ...
    ...

  In [SynchronizedBeforeSuite] at: k8s.io/kubernetes/test/e2e_node/e2e_node_suite_test.go:235 @ 09/20/25 14:52:44.204
------------------------------
[SynchronizedBeforeSuite] [FAILED] [122.848 seconds]
[SynchronizedBeforeSuite] 
k8s.io/kubernetes/test/e2e_node/e2e_node_suite_test.go:235

  [FAILED] SynchronizedBeforeSuite failed on Ginkgo parallel process #1
    The first SynchronizedBeforeSuite function running on Ginkgo parallel process
    #1 failed.  This suite will now abort.

  
  In [SynchronizedBeforeSuite] at: k8s.io/kubernetes/test/e2e_node/e2e_node_suite_test.go:235 @ 09/20/25 14:52:44.211
------------------------------

Summarizing 8 Failures:
  [FAIL] [SynchronizedBeforeSuite] 
  k8s.io/kubernetes/test/e2e_node/e2e_node_suite_test.go:235

  ...
  ...

Ran 0 of 876 Specs in 130.799 seconds
FAIL! -- A BeforeSuite node failed so all tests were skipped.

Ginkgo ran 1 suite in 2m11.106353619s

Test Suite Failed
F0920 14:52:52.136374   21331 run_local.go:101] Test failed: exit status 1
exit status 1
make: *** [Makefile:292: test-e2e-node] Error 1
```

I did a little bit looking around and realised that my test is not able to start `kubelet` properly.

But I still couldn't understand why exactly! Because the Kubelet binary is built as part of the test run and is available in the path - `_output/local/go/bin/kubelet`.  
(I didn't have to install kubelet binary by myself, it was built as part of the above make target run along with other components like kube-apiserver, etc.)

Some more digging (well, actually reading the above error message once again, *properly*).  
I realised, I can find out the actually issue in the kubelet logs.  

Below log helped:  

>  I0920 14:50:44.042571   22650 server.go:133] Output file for server "kubelet": /tmp/_artifacts/250920T145006/kubelet.log

So, checking the kubelet logs and it gave me the problem right away.

```bash
> tail -f /tmp/_artifacts/250920T145006/kubelet.log

I0920 14:50:44.151457   22917 server.go:579] "Sending events to api server"
I0920 14:50:44.151587   22917 swap_util.go:119] "Swap is on" /proc/swaps contents=<
	Filename				Type		Size		Used		Priority
	/dev/nvme0n1p3                          partition	2097472		0		-2
 >
E0920 14:50:44.151610   22917 run.go:72] "command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap or set --fail-swap-on flag to false"
```

The `swap` is ON, on my machine.  
The test setup expects me to to turn it OFF atleast for the duration of the execution of the test.

```bash

sudo swapoff -a
```

I turned off the swap and reran the test.

```bash
> KUBE_ROOT=$(pwd) make test-e2e-node FOCUS="just the scheduled condition set"

  << Captured StdOut/StdErr Output
------------------------------
SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS

Ran 1 of 876 Specs in 37.704 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 875 Skipped

Ginkgo ran 1 suite in 37.997204389s
Test Suite Passed
```

It worked! The test ran successfully! Hurrah!

But where to check the logs now?  
I can't see any test execution logs, information like Pod status (pending -> initializing -> running, etc).

Figured out - the `make test-e2e-node` run captures a bunch of artifacts from the execution of the test.

I can find them in the `/tmp/_artifacts` path.

```bash
❯ ls /tmp/_artifacts/250920T153001

build-log.txt  containerd-installation.log  containerd.log  junit_01.xml  kern.log  kubelet.log  services.log
```

And in my case, for the Pod status related information, I can check those from the Kubelet logs.

```bash
❯ cat /tmp/_artifacts/250920T153001/kubelet.log
```

So, here we go. I have a solution for running my new node e2e tests locally.

(Off I go to implement the test! o/)



