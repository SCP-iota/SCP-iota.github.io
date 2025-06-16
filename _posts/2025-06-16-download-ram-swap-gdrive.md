---
layout: blog_post
category: software
title: How to Actually Download More RAM - Google Drive as Linux Swap Space
description: 'What if you really could "download more RAM?" Linux allows any block device to be swap space, so maybe you can...'
---

We've all seen the jokes. "I can't play that game anymore because the update needs too much RAM." "Just download more lol." For those without ad blockers, promises of "free downloadable RAM" plague the Internet with false hope and countless strains of malware. Obviously it's impossible for real, physical RAM - the kind that has a direct line to your CPU - but you might not have known that it's possible to expand the software-level maximum memory capacity of your system with an online service. Before you get your hopes up, I must warn you - it's nowhere near efficient and probably not even worth doing. However, I'm not here to do things worth doing; I'm here to do what they said couldn't be done.

## "What sorcery is this?"

Most operating systems have a backup plan for when your physical RAM supply runs out: *swap space*. Unlike real RAM, this overflow space cannot be directly accessed by programs that are actively using memory; instead, when more memory is needed than can be allocated, pages of memory that haven't been used as recently are "swapped" out of physical RAM and stored in the swap space to allow that section of physical memory to be repurposed for the new demand. When a program tries to access the memory that was swapped out, the operating system will swap it back into physical RAM, which might require swapping out a different page. Typically, this extra space is provided by a portion of your hard drive space, since that's the second most accesible kind of memory your system has access to, but as we'll see, it doesn't have to be. Compared to ideal conditions of sufficient physical RAM, all this swapping can severly reduce efficiency, especially if programs need quick access to various pieces of data scattered around the memory address space, but at least it *works*, and that's more than can be said about `fatal error: runtime: out of memory`

## Loop the loop

On Linux, swap space is a dedicated type of partition separate from filesystems. For example, you might have a partition `/dev/sda3` that's formatted as a swap partition and configured to be your system's swap provider. In that case, `/dev/sda3` is a *block device* that the Linux kernel can read and write. You can create block devices out of anything, though - including regular files - using *loopback devices*. A loopback devices is a block device node that proxies read and write operations to a different path, which can be another device, a regular file, or a file provided by a virtual filesystem. To "download some RAM," we'll use the latter to put a swap partition on a file in cloud storage.

## Setup

### 1. Virtual Filesystem for Google Drive

This can be done with any cloud storage provider, but this guide will use Google Drive for the example. **I will warn that using cloud storage as swap space risks leaking sensitive data like passwords into the cloud. This is really just a silly guide for the lulz. Only do this at your own risk.**

To use Google Drive as a local filesystem, we need a virtual filesystem implementation for that. This guide will use [`google-drive-ocamlfuse`](https://github.com/astrada/google-drive-ocamlfuse), which has an installation guide [here](https://github.com/astrada/google-drive-ocamlfuse/wiki/Installation). For this guide, I have mounted my Google Drive into `~/GoogleDrive`.

It is important for the root user to have access to the filesystem. This is not the default for FUSE-based filesystems, so you'll need to uncomment the `user_allow_other` line in `/etc/fuse.conf` and add the `allow_root` option when mounting. For me, the command was:

```bash
google-drive-ocamlfuse -o=allow_root ~/GoogleDrive/
```

### 2. Create a Swap Space File and Loopback Device

To reserve space for swap, we need to create a file filled with the intended number of bytes. This example will allocate a gigabyte of swap.

```bash
dd bs=1024 count=1048576 if=/dev/zero of=~/GoogleDrive/swap
```

This command won't necessarily end automatically when the bytes have all been written, so you'll need to keep an eye on the file size to see when it is done. Use the file size shown by the virtual filesystem, not Google Drive's online details, since they may not synchronize immediately.

Now we need to create a loopback device that points to this file:

```bash
sudo losetup -f --show ~/GoogleDrive/swap
```

This command will output the path to the new loop device. For me, that was `/dev/loop0`, but it may be different for you. I'll store it in a variable and use only the variable to refer to it for now. The next step is formatting the content of the file as a swap partition:

```bash
SWAP_LOOP=/dev/loop0
sudo mkswap "$SWAP_LOOP"
```

## And We're Up!

Finally, we're ready to tell the Linux kernel to use the loop device as swap space:

```bash
sudo swapon "$SWAP_LOOP"
```

## Cleaning Up

When you're done using your terribly inefficient free downloadable RAM, you can stop using it as swap, remove the loop device, and unmount the virtual filesystem:

```bash
sudo swapoff "$SWAP_LOOP"
sudo losetup -d "$SWAP_LOOP"
fusermount -u ~/GoogleDrive
```

## Now Go Forth, and Engage the Trolls

With the knowledge in this guide, you are now ready to drop a link to this as a sarcastic response to all the bait and overused jokes about "downloading more RAM" you encounter during your online wanderings.