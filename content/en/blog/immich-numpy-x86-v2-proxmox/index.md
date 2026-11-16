---
title: "Immich machine learning won't start? Your Proxmox VM might be the culprit"
description: "The immich_machine_learning container crashes in a loop with « NumPy was built with baseline optimizations (X86_V2) ». Here is what causes it, and the five-minute fix."
date: 2026-11-16T10:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: ["immich", "proxmox", "self-hosting", "docker", "troubleshooting"]
---

I love Immich. Self-hosted, free and open source, it gives you full sovereignty over your photos and videos. Your data stays at home, on your server, with no middleman and no platform monetizing your memories. It is a remarkably polished project, and that is why watching image recognition break down is so frustrating: the software works perfectly, but one brick of your infrastructure is silently blocking it.

One morning, you open Immich. Semantic search returns nothing. Faces are no longer detected. You check the containers, and `immich_machine_learning` keeps restarting. In the logs, always the same message:


RuntimeError: NumPy was built with baseline optimizations:
(X86_V2) but your machine doesn't support:
(X86_V2).
```

That message is discouraging. It mentions Python, NumPy, compilation optimizations. The good news: the cause is not in Immich, and not in your Docker setup. It is in your Proxmox VM. And it takes five minutes to fix.

## Translating the error

NumPy says one simple thing: "I was compiled for the x86-64-v2 CPU level, and the machine running me does not have it".

The x86-64-v2 level is a set of instructions introduced around 2009: SSE4.2, POPCNT, CMPXCHG16B. Almost every x86 processor made since 2010 has them. That is why the message looks absurd. Your server is not fifteen years old.

Except the CPU NumPy sees is not your physical CPU. It is the CPU your hypervisor presents to the VM. And there, anything is possible.

## The cause: the virtual CPU type

Since version 3, Immich ships NumPy 2.x. The team states it in their [v3 migration guide](https://immich.app/blog/v3-migration): x86 CPUs must be at x86-64-v2 level or higher. It is not a bug, it is a documented requirement.

The problem happens on the hypervisor side. When you create a VM, Proxmox asks for a CPU type. If you left it at `kvm64`, or an old default of that kind, QEMU presents the guest with a generic processor from the early 2010s. That virtual CPU hides instructions like SSE4.2, even when your physical CPU has them.

The full path of the error:

1. Your physical CPU has SSE4.2. No problem there.
2. QEMU with type `kvm64` hides that instruction from the VM.
3. Docker inside the VM runs NumPy.
4. NumPy checks the visible CPU, does not find SSE4.2, refuses to start.

Your hardware is capable. Your VM is pretending not to be.

## The thirty-second diagnosis

Log into the VM that runs Immich, and run:

```bash
grep -o -E 'sse4_2|popcnt' /proc/cpuinfo | sort -u
```

Two possible outcomes:

- The command returns `popcnt` and `sse4_2`: the problem is elsewhere. Look at the image or the configuration.
- The command returns nothing: your virtual CPU is hiding the instructions. That is the case this tutorial covers.

To confirm the hardware is capable, run the same command on the Proxmox host, outside the VM:

```bash
grep -o 'sse4_2' /proc/cpuinfo | sort -u
```

If the host shows `sse4_2` and the VM does not, the diagnosis is certain.

## The fix

Five steps, and one precaution. Changing the CPU type only takes effect on a full VM start. A reboot from inside the guest OS is not always enough. Shut the VM down properly, or from Proxmox.

### 1. Stop the VM

In the Proxmox interface: right-click the VM, Shutdown. Or from the command line:

```bash
qm shutdown <vmid>
```

### 2. Change the CPU type

In the interface: VM → Hardware → Processors → Edit → Type, choose `host`. Or from the command line:

```bash
qm set <vmid> -cpu host
```

The `host` type exposes every instruction of the physical CPU to the guest, with no masking. The [Proxmox documentation](https://pve.proxmox.com/pve-docs/chapter-qm.html) recommends it for workloads that benefit from CPU instructions. Machine learning is a good example.

### 3. Restart and check the stack

Start the VM, then bring Immich back up:

```bash
docker compose up -d
```

Check the container logs:

```bash
docker logs immich_machine_learning --tail 50
```

You should see:

```
INFO     Initializing Immich ML [v3.4.1]
INFO     Starting gunicorn 26.2.0
INFO     Listening at: http://[::]:3003 (12)
INFO     Application startup complete.
```

No more NumPy errors. The container stays up.

### 4. Review CPU-related guest settings

After switching to `host`, the CPU the VM sees changes. If you tuned anything inside the guest (frequency governor, process pinning), this is the time to check those settings. Most of the time, there is nothing to do.

### 5. Restart the recognition jobs

Immich has not processed your photos during the outage. You need to catch up. In the admin interface:

- Administration → Jobs → Encode CLIP: rerun for semantic search and descriptions.
- Face Detection: rerun face detection.
- Face Recognition: rerun face recognition.

For Face Detection and Face Recognition, a reprocess option forces reprocessing of all media. Useful if detections failed silently across several versions.

## When `host` is not enough

One edge case remains. If the Proxmox host CPU itself does not support x86-64-v2, the `host` type will not help. Check with the `grep` command directly on the host. A processor older than 2010 will not pass the test.

In that case, the only option is pinning the `immich-machine-learning` image to a version from before the NumPy 2.x switch, a 2.7.x, until you replace the hardware. That is a workaround, not a solution. Modern Docker images keep tightening their CPU requirements, one after another.

## The broader lesson

This problem is not Immich-specific. NumPy 2.x requires x86-64-v2. RHEL 10 and its derivatives require x86-64-v3. The trend is clear: recently compiled software assumes your CPU is at least fifteen years old, not twenty.

A generic CPU type in a VM works fine for years. Then one day, a Docker image update crashes at startup, without you changing anything. The VM config has not moved since its creation.

The reflex to remember: whenever a container crashes at startup with a compiled-library error, check what CPU the VM actually sees. The command `grep -o -E 'sse4_2|popcnt|avx2' /proc/cpuinfo | sort -u` takes ten seconds. It tells you whether your hardware is guilty, or only your hypervisor.

## A final word for Immich

Immich is free, open source, and built by a small team that ships at an impressive pace. If the project serves you the way it serves me, consider supporting it through their [purchase page](https://immich.app/pricing). It is their support program, and contributors are recognized with a supporter badge in the app. A nice way to say thanks, and to fund the sovereignty it hands us for free.
```
