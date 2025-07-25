---
layout: post
title: "TIL: what is structural schema, and how to drop unknown fields in a custom resource (CR)"
tags: [kubernetes]
comments: false
---


Today, during our pair-(learning/programming) session for the [KEP 4595](https://github.com/kubernetes/enhancements/issues/4595) aka, CEL for CRD AdditionalPrinterColumns, [Sreeram](https://sreeram.xyz/) and I, came across this article – [Future of CRDs: Structural Schemas](https://kubernetes.io/blog/2019/06/20/crd-structural-schema/#towards-complete-knowledge-of-the-data-structure).  
(It's an old article from 2019, written by Dr. Stefan Schimanski[^2]. I will check if & what anything changed since 2019, but still reading this article in its current state, itself, was a turning point for me w.r.t my understanding of CRD(s).)

For the first time today, I understood - what is `structural schema` (something that I keep reading about and keep finding it refereneced everywhere within the kubernetes codebase, over and over again.)

So, **T**oday **I** **L**earnt (TIL)[^1]:

_**An OpenAPI v3 schema is a Structural Schema, if:**_

1. the core of an OpenAPI v3 schema, is made out of the following 7 constructs:
    - `properties`
    - `items`
    - `additionalProperties`
    - `type`
    - `nullable`
    - `title`
    - `descriptions`
  
  In addition, all types must be non-empty, and in each sub-schema only one of `properties`, `additionalProperties` or `items` may be used.

2. if all types are defined

3. the core is extended with value validation following the constraints:
    - (i) inside of value validations, there's no `additionalProperties`, `type`, `nullable`, `title`, `description`
    - (ii) all fields mentioned in value validation are specified in the core.

---

**Example of a structural schema** (understand below as a snippet from a CRD definition file, in the CRD `schema` field):  

  ```
  type: object
  properties:
    spec:
      type: object
      properties
        command:
          type: string
          minLength: 1                          # value validation
        shell:
          type: string
          minLength: 1                          # value validation
        machines:
          type: array
          items:
            type: string
            pattern: “^[a-z0-9]+(-[a-z0-9]+)*$” # value validation
      oneOf:                                    # value validation
      - required: [“command”]                   # value validation
      - required: [“shell”]                     # value validation
  required: [“spec”]                            # value validation
  ```

This schema is structural:
- because we have only used the permitted OpenAPI constructs (Rule 1).
- and each field (like `spec`, `command`, `shell`, `machines`, ...) has their types defined (Rule 2).
- plus, all value validations also follow the above defined rules (Rule 3).

---

**Example of a non-structural schema** (understand below as a snippet from a CRD definition file, in the CRD `schema` field):  
  
  ```
  properties:
    spec:
      type: object
      properties
        command:
          type: string
          minLength: 1
        shell:
          type: string
          minLength: 1
        machines:
          type: array
          items:
            type: string
            pattern: “^[a-z0-9]+(-[a-z0-9]+)*$”
      oneOf:
      - properties:
          command:
            type: string
        required: [“command”]
      - properties:
          shell:
            type: string
        required: [“shell”]
      not:
        properties:
          privileged: {}
  required: [“spec”]
  ```

This spec is non-structural for many reasons:
- `type:` object at the root is missing (Rule 2).
- inside of `oneOf` it is not allowed to use `type` (Rule 3-i).
- inside of `not` the property `privileged` is mentioned, but it is not specified in the core (Rule 3-ii).


---

The non-structural schema, highlighted a big problem (before we had structural schema in place), that is - if we can't use `not` to indicate the Kubernetes API server to drop the `privileged` field, then it will be forever preserved by the API server in Etcd.

This was fixed by **Pruning**. (which is not on by default in `apiextensions.k8s.io/v1` but had to be explicitly enabled in `apiextensions.k8s.io/v1beta1`).

Pruning in `apiextensions.k8s.io/v1beta1` is enabled via:

  ```
  apiVersion: apiextensions/v1beta1
  kind: CustomResourceDefinition
  spec:
    …
    preserveUnknownFields: false
  ```

--- 
---

[^1]: This's just me rephrasing, all what I learnt from and is mentioned in the original article – [Future of CRDs: Structural Schemas](https://kubernetes.io/blog/2019/06/20/crd-structural-schema/#towards-complete-knowledge-of-the-data-structure).  
[^2]: well, while searching for links to add as hyperlink for Dr. Stefan Schimanski, I came across more articles written by him, so, those are for my ToDos now.  
- [Kubernetes Deep Dive: Code Generation for CustomResources](https://www.redhat.com/en/blog/kubernetes-deep-dive-code-generation-customresources)
- [Kubernetes deep dive: API Server - part 1](https://www.redhat.com/en/blog/kubernetes-deep-dive-api-server-part-1)
- [Kubernetes Deep Dive: API Server – Part 2](https://www.redhat.com/en/blog/kubernetes-deep-dive-api-server-part-2)
- [Kubernetes Deep Dive: API Server – Part 3a](https://www.redhat.com/en/blog/kubernetes-deep-dive-api-server-part-2)
