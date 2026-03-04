---
layout: post
category: perf
---

Using `mmap(2)`, it is possible to remap sections of your executable to be backed by huge pages. Unfortunately, `perf record` doesn't play well when `/proc/self/maps` inevitably gets destroyed by your `mmap` shenanigans... here is how to make your symbols available to perf so you don't lose function attribution when generating flame graphs from perf data.

Here is an embedded flame graph (hopefully!): {% include svg.html asset="/assets/images/cpu-mysql-updated.svg" width=900 height=522 %}
