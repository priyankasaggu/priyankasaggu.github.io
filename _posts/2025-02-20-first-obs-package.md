---
layout: post
title: "The first package, I packaged in OBS – python-durationpy"
tags: [openSUSE]
comments: false
---



Just a small post to leave myself a memory. 🙂

[python-durationpy](https://build.opensuse.org/package/show/devel:kubic/python-durationpy) is the very first rpm package, I packaged all from scratch[^1] for the first time in OBS. 

This was needed to bump the [python-kubernetes](https://build.opensuse.org/package/show/devel:kubic/python-kubernetes) (a newly added python dependency) version to `31.0.0`.



---

[^1]: Well, not all from scratch. I took help from the existing [fedora rpm package](https://download.copr.fedorainfracloud.org/results/@fedora-review/fedora-review-2314132-python-durationpy/fedora-40-x86_64/08209771-python-durationpy/python-durationpy.spec).
