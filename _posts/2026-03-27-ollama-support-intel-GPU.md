---
layout: post
title: "ollama with Intel GPU"
---

## ollama + Intel GPU
Just heard that Ollama now supports Intel GPUs, and I happened to have an A380 to try it out;
the configuration is very simple: add `Environment="OLLAMA_VULKAN=1"` in /etc/systemd/system/ollama.service

```
jiyin@x99i:~$ sudo vim /etc/systemd/system/ollama.service
# add Environment="OLLAMA_VULKAN=1" in [Service] block

jiyin@x99i:~$ sudo systemctl daemon-reload
jiyin@x99i:~$ sudo systemctl restart  ollama
jiyin@x99i:~$ systemctl show ollama.service | grep -i env
Environment=PATH=/home/jiyin/.local/bin:/home/jiyin/bin:/usr/lib64/ccache:/usr/local/bin:/usr/bin OLLAMA_VULKAN=1
jiyin@x99i:~$ systemctl cat ollama.service | grep -i env
Environment="PATH=/home/jiyin/.local/bin:/home/jiyin/bin:/usr/lib64/ccache:/usr/local/bin:/usr/bin"
Environment="OLLAMA_VULKAN=1"
```

check gpu info from `systemctl status ollma`  
```
jiyin@x99i:~$ systemctl status ollama | cat 
● ollama.service - Ollama Service
     Loaded: loaded (/etc/systemd/system/ollama.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Fri 2026-03-27 18:00:44 HKT; 8min ago
 Invocation: 5126a117ba264d498338a979ec811d6e
   Main PID: 2012 (ollama)
      Tasks: 15 (limit: 154293)
     Memory: 234.8M (peak: 266.9M)
        CPU: 536ms
     CGroup: /system.slice/ollama.service
             └─2012 /usr/local/bin/ollama serve

3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.502+08:00 level=INFO source=routes.go:1729 msg="Ollama cloud disabled: false"
3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.559+08:00 level=INFO source=images.go:477 msg="total blobs: 21"
3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.560+08:00 level=INFO source=images.go:484 msg="total unused blobs removed: 0"
3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.561+08:00 level=INFO source=routes.go:1782 msg="Listening on 127.0.0.1:11434 (version 0.18.2)"
3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.564+08:00 level=INFO source=runner.go:67 msg="discovering available GPUs..."
3月 27 18:00:45 x99i.test.net ollama[2012]: time=2026-03-27T18:00:45.565+08:00 level=INFO source=server.go:430 msg="starting runner" cmd="/usr/local/bin/ollama runner --ollama-engine --port 44811"
3月 27 18:00:46 x99i.test.net ollama[2012]: time=2026-03-27T18:00:46.016+08:00 level=INFO source=server.go:430 msg="starting runner" cmd="/usr/local/bin/ollama runner --ollama-engine --port 41703"
3月 27 18:00:46 x99i.test.net ollama[2012]: time=2026-03-27T18:00:46.199+08:00 level=INFO source=server.go:430 msg="starting runner" cmd="/usr/local/bin/ollama runner --ollama-engine --port 41059"
3月 27 18:00:47 x99i.test.net ollama[2012]: time=2026-03-27T18:00:47.663+08:00 level=INFO source=types.go:42 msg="inference compute" id=8680a556-0500-0000-0400-000000000000 filter_id="" library=Vulkan compute=0.0 name=Vulkan0 description="Intel(R) Arc(tm) A380 Graphics (DG2)" libdirs=ollama,vulkan driver=0.0 pci_id=0000:04:00.0 type=discrete total="5.9 GiB" available="5.4 GiB"
3月 27 18:00:47 x99i.test.net ollama[2012]: time=2026-03-27T18:00:47.663+08:00 level=INFO source=routes.go:1832 msg="vram-based default context" total_vram="5.9 GiB" default_num_ctx=4096
jiyin@x99i:~$
jiyin@x99i:~$ systemctl status ollama | grep A380 
3月 27 18:00:47 x99i.test.net ollama[2012]: time=2026-03-27T18:00:47.663+08:00 level=INFO source=types.go:42 msg="inference compute" id=8680a556-0500-0000-0400-000000000000 filter_id="" library=Vulkan compute=0.0 name=Vulkan0 description="Intel(R) Arc(tm) A380 Graphics (DG2)" libdirs=ollama,vulkan driver=0.0 pci_id=0000:04:00.0 type=discrete total="5.9 GiB" available="5.4 GiB"
```
