---
layout: post
title: "RDMA hardware troubleshooting note(rdma硬件折腾记)"
---

RHEL-10 has **again** phased out our older InfiniBand (IB) cards used for RDMA testing, 
with the minimum supported model now being MLX5.  

While discussing funding for new cards, I was surprised to discover that a batch of 
new servers already have **onboard MLX5 interfaces** on their motherboards.  

And contacted IT colleagues and borrowed IB cables from the network-qe team to connect 
these interfaces to the nearest IB switch.  
However, `ibstat` showed the `link_layer` as Ethernet. Consulting with storage team 
colleagues revealed that MLX5 interfaces typically support dual protocol (IB/Ethernet), 
and the `link_layer` can be modified via software tools:  
```
   dnf install mstflint  
   mstconfig -d mlx5_0 set LINK_TYPE_P1=1  
```  

Attempts with both open-source drivers and Mellanox drivers **all failed**, and 
we eventually discovered the onboard interface only supports **Ethernet** :(  

After contacting IT again and switching to Ethernet SFP+ cables, the interface still 
wouldn't come up. Further confirmation revealed the connected IB switch has **no 
management port** and only supports **IB protocol**, not Ethernet.  
Both sides are **incompatible**...  

Currently, IT colleagues are searching for older switches that support Ethernet SFP+ interfaces, 
but there's no progress yet.
