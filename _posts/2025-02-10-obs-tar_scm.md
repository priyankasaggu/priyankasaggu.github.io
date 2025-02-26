---
layout: post
title: "OBS tar_scm service, how to include .git in output tarball"
tags: [openSUSE]
comments: false
---


How to also include the `.git`[^1] directory in the output tarball created by the `tar_scm` OBS (OpenSUSE Build Service) service module?

Use the service parameter – [package-meta](https://github.com/openSUSE/obs-service-tar_scm/blob/b49251606834eda4ccc84d93d68917617eef254f/tar_scm.service.in#L167-L170).

  ```
    <parameter name="package-meta">
      <description>Package the metadata of SCM to allow the user or OBS to update after un-tar. Please be aware that this parameter has precedence over the "exclude" paramter.</description>
      <allowedvalue>yes</allowedvalue>
    </parameter>
  ```

Following is an example of an OBS `_service` file (where I needed to use the above parameter):

```
<services>
  <service name="tar_scm" >
    <param name="url">https://github.com/etcd-io/etcd.git</param>
    <param name="scm">git</param>
    <param name="revision">v3.5.16</param>
    <param name="package-meta">yes</param>
    <param name="versionformat">@PARENT_TAG@</param>
    <param name="versionrewrite-pattern">v(.*)</param>
  </service>
  <service name="recompress">
    <param name="file">*.tar</param>
    <param name="compression">gz</param>
  </service>
  <service name="go_modules">
    <param name="archive">*.tar.gz</param>
  </service>
</services>
```

The `_service` file contains three OBS service modules:
- `tar_scm`[^2]: to clone a github repo (in this case, it's cloning the etcd repo code on tag v3.5.16), and then compress the cloned source code directory to a tarball (*.tar).
- `recompress`[^3]: recompress the above `*.tar`tarball to `*.tar.gz`
- `go_modules`[^4]: find the above source tarball, untar and cd into it, and run in sequence - `go mod download && go mod verify && go mod vendor` and give back a tarball `vendor.tar.gz` with compressed content from `vendor/` directory containing the vendored go module dependencies needed by the go project.

---

[^1]: `.git` – the directory containing the metadata and object database of the git repository
[^2]: source: [tar_scm](https://github.com/openSUSE/obs-service-tar_scm) 
[^3]: source: [recompress](https://github.com/openSUSE/obs-service-recompress)
[^4]: source: [go_modules](https://github.com/openSUSE/obs-service-go_modules)

