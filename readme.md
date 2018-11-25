# Ubuntu Full Disk Encryption with BTRFS
by Stewart Gebbie
2018

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

__HERE BE DRAGONS__ please take anything said below with a pinch of salt and
follow at your own risk. I have not double checked all the details and these
amendments to the original guide have not been battle tested on other system
configurations.

## Partitioning

Given a laptop that is pre-installed with Windows, one does not necessarily want
to format the whole drive. So shrinking the Windows C: is a good option.
However, this will generally cause one to bump into the problem of unmovable
files that prevents one from shrinking the partition as much as is desired.
The following guide:

- ["How to shrink a disk volume beyond the point where any unmovable files are located by Mihai Neacsu"][shrinking-past-unmovable]

provides a concise list of temporary tweaks that will make it possible to shrink
the partition to closer to the actual usage limits. It does this by:

- removing hiberfil.sys
- removing pagefile.sys
- removing swapfile.sys
- disabling rollback checkpoints
- disabling diagnostic memory dumps

All of these can be re-enabled again once the partition has been resized.

## Secure Boot

While Secure Boot may have its security benefits it certainly does tend to cause
some problems. In the case of GRUB this turned out to be a rather silent issue
that causes boot up, post installation, to fail.

The problems seems to be that while there are appropriately signed versions of
GRUB, this version does not actually include all needed modules. In particular
it does not include the `luks` module. To compound the problem this does not
actually display an error. Instead, GRUB simply never prompts for the LUKS
password.

Running `grub> lsmod` eventually shows the problem because it is evident that
the luks module is missing. However executing `grub> insmod luks` doesn't help
since the signed version does not seem to allow for module loading. But, it also
simply drops through silently without displaying a failure.

In short, it is necessary to disable "Secure Boot."

Following this the boot process will do "the right thing" (TM) and prompt for a
password:

```
Attempting to decrypt master key...
Enter passphrase for hd?,gpt? (...):
Slot 0 opened
```

## Hybrid Suspend

The Lenovo ThinkPad X1 Carbon 6th Generation that I was installing Ubuntu on,
with full system encryption, still had a bug that prevented Linux from making
use of the S3 sleep state.

This has been fixed in later versions of the BIOS:
   TODO include link

Simply upgrade to a newer BIOS version and then remember to toggle the power
management sleep mode from "Windows 10" to "Linux."

## BTRFS

I wanted to rather use BTRFS as my main filesystem. This will enable me to
create and destroy subvolumes without making using of LVM.

I originally thought it would be a simple matter of swapping out `mkfs.ext4` for
`mkfs.btrfs`. But, as always, the devil is in the details.

There are two related problems that show up due to the use of subvolumes. In
particular, when Ubuntu performs an installation on BTRFS it creates two
"magical" subvolumes be default:

- `@` - used as the root subvolume to be mounted on `/`
- `@home` - used as the "user" subvolume and mounted on `/home`

The problem is that the `encryptinstallation` script is not aware of these
changes. During the script execution one of the steps is to create a `chroot`
environment. For this, it simply mounts the `system` partition into the
temporary `root/` (it will also optionally mount a `data` partition onto
`root/home/` if chosen). But this script now needs to be amended to:

- mount `/dev/mapper/system-root` on `/` with the additional mount option
  `-o subvol=@`

and then (even when no explicit data partition has been selected):

- explicitly mount `/dev/mapper/system-root` on `/home` with the mount option
  `-o subvol=@home`

For sanity sake, also double check that the final `grub.cfg` includes the
command line kernel option: `rootflags=subvol=@`.

## LUKS Unlock

With all of the above issues sorted out the system will not boot. However, the
boot sequence is surprising slow. More specifically, depending on your hardware
combination you might find that GRUB seems to hang for over a minute before
proceeding to boot Linux.

It turns out that this is by design - sort of ;)

This is actually a side effect of LUKS key handling, and in particular PBKDF2.

As part of using a "Password Based Key Derivation Function" the password is
iteratively hashed in order to improve security against brute force attacks.

Unfortunately the GRUB cryptographic routines are not optimised in the same way
as the `cryptsetup` implementations. Apparently the GRUB versions either can
not, or do not, make use of the AES instructions and other 64 Bit instructions.
As a result the GRUB implementation is easily and order of magnitude slower that
the `cryptsetup` implementation.

The means that when a password is validated against the key slot it takes GRUB a
lot longer to first perform the necessary number of iterations.

On my machine this stretched the validate from 1-2 seconds to ~70s.
Unfortunately I found this to be simply too long when booting up.

The solution is to manually reset the key and force a lower iteration count.

__WARNING__ this will weaken the security against brute force attacks.

I tuned my LUKS unlock time down to around 10 seconds in GRUB.

### Creating new keys

Note, GRUB will iterate over each key slot sequentially. As such, if you have
more than one key slot, then it is useful to make sure that the keys that are
going to be used by GRUB are given lower key slot indicies.

Newer versions of `cryptsetup` include a function: `luksConvertKey`. However,
the default version in Ubuntu 18.04 LTS is 2.0.2 and does not yet include this
function. As such, this guide uses `luksAddKey`. But, as said, it is important
that the lower iteration count key also has a lower key slow index. So we do
following (after potentially using `luksHeaderBackup` first; paying attention to
warnings regarding making copies of key material):

```
echo Add an extra key for use when removing and adding to slot 0
echo (A lower iteration count is not needed at this point in time)
echo (Provide the passphrase for the exiting slot 0)
cryptsetup luksAddKey /dev/nvme0n1p?
echo Now remove key slot 0
echo (This is selected based on the first matching password.)
echo (So enter the original passphrase and not the value entered above, if they differ)
cryptsetup luksRemoveKey /dev/nvme0n1p?
echo Now re-add a key that will then fill slot 0, but with a tuned iteration count
cryptsetup luksAddKey --pbkdf-force-iterations 50000 /dev/nvme0n1p?
```

At this point in time it is probably work performing a simply sanity check and
dump the header just to check the layout:

```
cryptsetup luksDump /dev/nvme0n1p5
```
Confirm that `Key Slot 0` is `ENABLED` and that the `Iterations:` value matches
the number provided with `--pbkdf-force-iterations` value above.

### Caveat

Be careful not to remove the "internal" key slot set up by
`encryptinstallation`. This corresponds to the key material stored in
`/etc/crypt.system` and is used to prevent asking for the key twice by having
the `initramfs` image use this key instead.

However, if you remove it you can break your boot sequence...

No prizes for guessing how I realised that ;)

# Summary

Thanks again to Paddy Landau for the original guide and scripts. These are high
quality and robust scripts, with great source code documentation.

However, if you wish to set up full disk/system encryption with the addition of
BTRFS then hopefully the above details will help you on you way.


[manual-full-system-encryption]: https://help.ubuntu.com/community/ManualFullSystemEncryption "Manual Full System Encryption by Paddy Landau"
[shrinking-past-unmovable]: https://www.download3k.com/articles/How-to-shrink-a-disk-volume-beyond-the-point-where-any-unmovable-files-are-located-00432 "How to shrink a disk volume beyond the point where any unmovable files are located by Mihai Neacsu on 25 June 2014"
