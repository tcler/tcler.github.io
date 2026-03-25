---
layout: post
title: "fedora install pkg from rawhide"
---

## update rocm from rawhide
rocm-7.1 on fedora-44 does not work with ollama on AMD max395 platform,  
how to install rocm-7.2 from rawhide?

```
sudo yum install -y fedora-repos-rawhide
sudo yum install -y --enablerepo rawhide rocm
sudo yum update -y --enablerepo rawhide rocm
```
