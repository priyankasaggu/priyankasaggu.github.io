---
layout: post
title: "Access data persisted in Etcd with etcdctl and kubectl"
tags: [kubernetes]
comments: false
---

I created the following CRD (Custom Resource Definition) with — `kubectl apply -f crd-with-x-validations.yaml`:

``` 
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must be in the form: <plural>.<group>
  name: myapps.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: example.com
  scope: Namespaced
  names:
    # kind is normally the CamelCased singular type. 
    kind: MyApp
    # singular name to be used as an alias on the CLI
    singular: myapp
    # plural name in the URL: /apis/<group>/<version>/<plural>
    plural: myapps
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            x-kubernetes-validations: 
              - rule: "self.minReplicas <= self.maxReplicas"
                messageExpression: "'minReplicas (%d) cannot be larger than maxReplicas (%d)'.format([self.minReplicas, self.maxReplicas])"
            type: object
            properties:
              minReplicas:
                type: integer
              maxReplicas:
                type: integer
```


I want to check how the above CRD is persisted in Etcd.

I have two ways to do the job:

#### Option 1:

Use `etcdctl` to directly verify the persisted data in Etcd.[^1]

My three steps process:
- Exec inside the etcd pod in the `kube-system` namespace of your kubernetes cluster —
   `kubectl exec -it -n kube-system etcd-kep-4595-cluster-control-plane -- /bin/sh`
- Create alias —
   `alias e="etcdctl --endpoints 127.0.0.1:2379 --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt"`
- Access the data —
   `e get --prefix /registry/apiextensions.k8s.io/`

```
sh-5.2# e get --prefix /registry/apiextensions.k8s.io/

/registry/apiextensions.k8s.io/customresourcedefinitions/shirts.stable.example.com
{"kind":"CustomResourceDefinition","apiVersion":"apiextensions.k8s.io/v1beta1","metadata":{"name":"shirts.stable.example.com","uid":"09696eb0-d58b-4a21-8820-b2230b13707e","generation":1,"creationTimestamp":"2025-02-21T12:38:19Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"apiextensions.k8s.io/v1\",\"kind\":\"CustomResourceDefinition\",\"metadata\":{\"annotations\":{},\"name\":\"shirts.stable.example.com\"},\"spec\":{\"group\":\"stable.example.com\",\"names\":{\"kind\":\"Shirt\",\"plural\":\"shirts\",\"shortNames\":[\"shrt\"],\"singular\":\"shirt\"},\"scope\":\"Namespaced\",\"versions\":[{\"additionalPrinterColumns\":[{\"jsonPath\":\".spec.color\",\"name\":\"Fruit\",\"type\":\"string\"}],\"name\":\"v1\",\"schema\":{\"openAPIV3Schema\":{\"properties\":{\"spec\":{\"properties\":{\"color\":{\"type\":\"string\"},\"size\":{\"type\":\"string\"}},\"type\":\"object\"}},\"type\":\"object\"}},\"served\":true,\"storage\":true}]}}\n"},"managedFields":[{"manager":"kube-apiserver","operation":"Update","apiVersion":"apiextensions.k8s.io/v1","time":"2025-02-21T12:38:19Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:acceptedNames":{"f:kind":{},"f:listKind":{},"f:plural":{},"f:shortNames":{},"f:singular":{}},"f:conditions":{"k:{\"type\":\"Established\"}":{".":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}},"k:{\"type\":\"NamesAccepted\"}":{".":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}}}},"subresource":"status"},{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"apiextensions.k8s.io/v1","time":"2025-02-21T12:38:19Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:conversion":{".":{},"f:strategy":{}},"f:group":{},"f:names":{"f:kind":{},"f:listKind":{},"f:plural":{},"f:shortNames":{},"f:singular":{}},"f:scope":{},"f:versions":{}}}}]},"spec":{"group":"stable.example.com","version":"v1","names":{"plural":"shirts","singular":"shirt","shortNames":["shrt"],"kind":"Shirt","listKind":"ShirtList"},"scope":"Namespaced","validation":{"openAPIV3Schema":{"type":"object","properties":{"spec":{"type":"object","properties":{"color":{"type":"string"},"size":{"type":"string"}}}}}},"versions":[{"name":"v1","served":true,"storage":true}],"additionalPrinterColumns":[{"name":"Fruit","type":"string","JSONPath":".spec.color"}],"conversion":{"strategy":"None"},"preserveUnknownFields":false},"status":{"conditions":[{"type":"NamesAccepted","status":"True","lastTransitionTime":"2025-02-21T12:38:19Z","reason":"NoConflicts","message":"no conflicts found"},{"type":"Established","status":"True","lastTransitionTime":"2025-02-21T12:38:19Z","reason":"InitialNamesAccepted","message":"the initial names have been accepted"}],"acceptedNames":{"plural":"shirts","singular":"shirt","shortNames":["shrt"],"kind":"Shirt","listKind":"ShirtList"},"storedVersions":["v1"]}}
```


#### Option 2:

Use `kubectl` to access the persisted data from Etcd –  

`kubectl get --raw /apis/apiextensions.k8s.io/v1/customresourcedefinitions/shirts.stable.example.com`

```
> kubectl get --raw /apis/apiextensions.k8s.io/v1/customresourcedefinitions/shirts.stable.example.com

{"kind":"CustomResourceDefinition","apiVersion":"apiextensions.k8s.io/v1","metadata":{"name":"shirts.stable.example.com","uid":"09696eb0-d58b-4a21-8820-b2230b13707e","resourceVersion":"594","generation":1,"creationTimestamp":"2025-02-21T12:38:19Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"apiextensions.k8s.io/v1\",\"kind\":\"CustomResourceDefinition\",\"metadata\":{\"annotations\":{},\"name\":\"shirts.stable.example.com\"},\"spec\":{\"group\":\"stable.example.com\",\"names\":{\"kind\":\"Shirt\",\"plural\":\"shirts\",\"shortNames\":[\"shrt\"],\"singular\":\"shirt\"},\"scope\":\"Namespaced\",\"versions\":[{\"additionalPrinterColumns\":[{\"jsonPath\":\".spec.color\",\"name\":\"Fruit\",\"type\":\"string\"}],\"name\":\"v1\",\"schema\":{\"openAPIV3Schema\":{\"properties\":{\"spec\":{\"properties\":{\"color\":{\"type\":\"string\"},\"size\":{\"type\":\"string\"}},\"type\":\"object\"}},\"type\":\"object\"}},\"served\":true,\"storage\":true}]}}\n"},"managedFields":[{"manager":"kube-apiserver","operation":"Update","apiVersion":"apiextensions.k8s.io/v1","time":"2025-02-21T12:38:19Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:acceptedNames":{"f:kind":{},"f:listKind":{},"f:plural":{},"f:shortNames":{},"f:singular":{}},"f:conditions":{"k:{\"type\":\"Established\"}":{".":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}},"k:{\"type\":\"NamesAccepted\"}":{".":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}}}},"subresource":"status"},{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"apiextensions.k8s.io/v1","time":"2025-02-21T12:38:19Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:conversion":{".":{},"f:strategy":{}},"f:group":{},"f:names":{"f:kind":{},"f:listKind":{},"f:plural":{},"f:shortNames":{},"f:singular":{}},"f:scope":{},"f:versions":{}}}}]},"spec":{"group":"stable.example.com","names":{"plural":"shirts","singular":"shirt","shortNames":["shrt"],"kind":"Shirt","listKind":"ShirtList"},"scope":"Namespaced","versions":[{"name":"v1","served":true,"storage":true,"schema":{"openAPIV3Schema":{"type":"object","properties":{"spec":{"type":"object","properties":{"color":{"type":"string"},"size":{"type":"string"}}}}}},"additionalPrinterColumns":[{"name":"Fruit","type":"string","jsonPath":".spec.color"}]}],"conversion":{"strategy":"None"}},"status":{"conditions":[{"type":"NamesAccepted","status":"True","lastTransitionTime":"2025-02-21T12:38:19Z","reason":"NoConflicts","message":"no conflicts found"},{"type":"Established","status":"True","lastTransitionTime":"2025-02-21T12:38:19Z","reason":"InitialNamesAccepted","message":"the initial names have been accepted"}],"acceptedNames":{"plural":"shirts","singular":"shirt","shortNames":["shrt"],"kind":"Shirt","listKind":"ShirtList"},"storedVersions":["v1"]}}
```

---
---


[^1]: I realised while I'm accessing the same CRD data with `etcdctl` and `kubectl`, I'm getting a few things different in my output. In case of `etcdctl` — I get **(i)** `"version":"v1"`, **(ii)** the CRD `schema` is stored in field `"validation":{"openAPIV3Schema":{"type":"object","properties":{"spec":{"type":"object","properties":{"color":{"type":"string"},"size":{"type":"string"}}}}}}` and **(iii)** and there's a top level `additionalPrinterColumns`. While in case of `kubectl`— I don't get the above bits, and instead I get both, the `schema` and the `additionalPrinterColumns` stored in the `versions` array - `"versions":[{"name":"v1","served":true,"storage":true,"schema":{"openAPIV3Schema":{"type":"object","properties":{"spec":{"type":"object","properties":{"color":{"type":"string"},"size":{"type":"string"}}}}}},"additionalPrinterColumns":[{"name":"Fruit","type":"string","jsonPath":".spec.color"}]}]`.  This is (maybe) something to do with how currently (as of writing) Kubernetes stores/persists CRD `v1` as `v1beta1` in Etcd, because `v1` takes more space to represent the same CRD (due to denormalization of fields among multi-version CRDs) and we have CRDs in the wild that are already bumping against the max allowed size (Thank you, [Jordan Liggit](https://jordan.liggitt.net/) for explaining this.) Read this[^2] and this[^3] for some context.
[^2]: [https://github.com/kubernetes/kubernetes/blob/931ad2a9fdedaf1e47126f5b3e5880eb3708bfb2/pkg/controlplane/apiserver/apiextensions.go#L55-L56](https://github.com/kubernetes/kubernetes/blob/931ad2a9fdedaf1e47126f5b3e5880eb3708bfb2/pkg/controlplane/apiserver/apiextensions.go#L55-L56)
[^3]: Attempt to bump the storage version from v1beta1 → v1, but was blocked on [https://github.com/kubernetes/kubernetes/issues/82292](https://github.com/kubernetes/kubernetes/issues/82292) 
