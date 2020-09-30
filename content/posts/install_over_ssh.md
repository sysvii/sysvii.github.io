# Installing Arch Linux over SSH

## Notes

# Having a go

`cp -r /usr/share/archiso/configs/baseline archiso`

```bash
> find .
.
./build.sh
./isolinux
./isolinux/isolinux.cfg
./mkinitcpio.conf
./syslinux
./syslinux/syslinux.cfg
```

Looking for packages.x86_64 :thinking:

```
> rg "packages"

# nothing ?!
```

Ah I got the wrong one! My desire for a bare bones build dashed

round 2

`cp -r /usr/share/archiso/configs/releng archiso`

```bash
> find .
.
./pacman.conf
./efiboot
./efiboot/...
./airootfs
./airootfs/...
./build.sh
./isolinux
./isolinux/isolinux.cfg
./mkinitcpio.conf
./syslinux
./syslinux/...
./packages.x86_64 # THERE IT IS!!
```

That's more I like it!

Time to add what I need to add `sshd` to remote into, but what package is it
again?

```
> pacman -Qo `which sshd`
/usr/bin/sshd is owned by openssh 8.3p1-3
```

```bash
> rg 'openssh'
packages.x86_64
51:openssh
```

Oh it is already there, that's kewl with me

Let's see what the docs say next

configure the script

> thoughts on the script 

Do the iso build 

image it to a USB

boot the thing

find it on the network

nmap hacker time

```
watch nmap -sn 192.168.1.0/24
```

Is it even coming up?

ugh, this took ages & ended up borrowing my flatmate's monitor to figure out
wtf went wrong

It worked in the end & I'm not sure why

## Context

why: cause I don't got a VGA port on my monitor (well my flatmate does but I
don't wanna take their monitor for a day)

the machine: ol' HP home server, lotta disk, BIGDISK

## Resources

https://wiki.archlinux.org/index.php/Install_Arch_Linux_via_SSH
https://wiki.archlinux.org/index.php/Archiso
https://bbs.archlinux.org/viewtopic.php?id=90635
