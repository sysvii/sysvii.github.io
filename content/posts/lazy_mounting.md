+++
title = "Lazy mounting a disk"
description = "Usage based mounting"
date = 2018-04-23
updated = 2020-09-26

[taxonomies]
categories = ["tech"]
tags = ["systemd", "fstab", "linux"]
+++

## Background

Having a big noisy spinning disk only for storing big data, like _linux iso's..._,
is becoming the norm. However, I don't want it on all the time for the
sake of that sweet SSD zen.

If only there was something that would only mount a disk when it is needed!!

## systemd magic sauce



## `/etc/fstab` config

```
UUID=0000000000000000   /mnt/disk        ntfs-3g    rw,noauto,x-systemd.automount  0   0
```

## Let's break it down

The configuration above is for `/etc/fstab`. 

Most of this is coming from [`man fstab`](https://www.man7.org/linux/man-pages/man5/fstab.5.html) (section 5? I still don't really get `man` sections).

Each whitespace separated section is a field for an `fstab` entry.

### `UUID=0000000000`

`fstab` docs calls this the `fs_spec` field. This is points to where the disk
is.

The classic example is `/dev/sda1` to point to a physical disk.
This is a big draw back cause what physical disk behind `/dev/sda` could change
to a range of reason (kernel ordering, changing around cables, etc.) so this is
not a particularly stable method.

Smarter people than me realised this ages ago & let you use disk unique
identifiers like the `LABEL` from disk partitions or the `UUID` of the
partitions.

`UUID` being the most stable, mainly cause it takes some effort to get a
collision. But, that does leave you in a mess of trying to find what that
special `UUID` for you is.

Thankfully those smart people thought of that & we got `lsblk`.
It can tell you a lot about drives & partitions on your computer, but for now
there is just two parts of it relevant (& I half know how to use).

You can get the basic list of what is mounted where (super helpful to just know
what is about & where you mounted that weird USB drive)

```bash
> lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   1.8T  0 disk
├─sda1   8:1    0   1.8T  0 part /
└─sda2   8:2    0    33G  0 part [SWAP]
```

To get particular info about a device you can `-o <OPTION>` to output an `OPTION`.
In our case we want the `UUID` attribute, so we use:

```bash
> lsblk -o UUID /dev/sda1
UUID
0000000000000000
```

### `/mnt/disk`

The `fstab` calls this the `fs_file`, or where the mount point target.

There isn't much more to say than it describes where in the file system it will
be mounted to.

### `nfts-3g`

Following where, it is how to read what is being mount. It is the type of
file system.

In this case it is a non-default format of [`ntfs-3g`](https://wiki.archlinux.org/index.php/NTFS-3G), aka the current driver
for the windows friendly format [NTFS](https://en.wikipedia.org/wiki/NTFS).

### `rw,noauto,x-systemd.automount`

Next is the real magic sauce, the mount options!

> Caller 1: Hey, I know `mount`, is this like the normal one?
> Caller 2: Yeah, it is actually is the same options passed to `mount`!
> Caller 1: Does that mean what I put in `/etc/fstab` work with `mount -o <OPTIONS>`?!
> Caller 2: Why yes it does!!
> Caller 1: ![Animated GIF of an over shocked person from the 1980's](http://c0.thejournal.ie/media/2014/04/shocked-5.gif "I am shocked as you are")
> Caller 2: It is a handy way to test `fstab` changes without having to get a recovery device out

### Standard Options

#### `rw`

Mount the filesystem in read-write mode, handy if you want to change things.

> Note: `ro` is for ready-only in case you ever need it.


#### `uid=1XXX,gid=1XXX,umask=022`

This is more just for `ntfs-3g`, but it is to set the disk permissions for the
drive. 

### Magic Systemd options

#### `noauto`

Don't do the normal thing at boot & mount that ~~floppy~~ drive.

#### `x-systemd.automount`

A systemd extension that will mount a pseudo-filesystem to your mount point
that will get swapped out by your real one once someone tries to access
anything in it.

It is like those sets that are just big façades, but once you know on the door
the set crew has already set up the interior for you (albeit maybe with some awkward smiling to the camera as the ~~disk loads~~ set is built)

More can details be found with [`man systemd-mount`](https://man7.org/linux/man-pages/man1/systemd-mount.1.html)

### `0  0`

These last two are a bit of a mystery to me, though the great `man fstab` tells
me these are controls for `dump` & `fsck` respectively.


## Conclusion


