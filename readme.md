# Ubuntu Full Disk Encryption with BTRFS

This is based on the excellent set of instructions and installations scripts
that can be found via: [Manual Full System Encryption][manual-full-system-encryption].

This does not attempt to provide a nice neat set of scripts to automate the
installation for BTRFS, but rather enumerates my various tweaks. It also
includes some tweaks for issues related to boot time and partitioning.

In particular I will cover the rough steps taken to:

1. allow better shrinking of existing Windows partitions
2. fix boot bug due to Secure Boot and GRUB modules
4. tweak BIOS sleep state to support hybrid-suspend (TODO)
3. tweak creation scripts to support BRTFS
4. change LUKS keys to reduce boot time (warning)

## Partitioning

## Secure Boot

## Hybrid Suspend

## BTRFS

## LUKS Unlock


[manual-full-system-encryption]: https://help.ubuntu.com/community/ManualFullSystemEncryption "Manual Full System Encryption by Paddy Landau"
