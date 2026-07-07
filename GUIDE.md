# NixOS Bootstrapping Guide

A step-by-step guide to install this configuration on a **new machine**, from
downloading the ISO to running the full flake.

This guide supersedes the terse notes in [`README.md`](README.md),
[`notREADME.md`](notREADME.md) and [`disko.md`](disko.md); those are kept as
raw references. Everything here has been checked against the actual repository
(`disko` layout, host wiring, `sops` keys, `git.nix`, `justfile`).

---

## 0. How the bootstrap works (read this first)

Installing the *entire* configuration in one shot on a fresh, possibly weak
machine is slow and error-prone. Instead we use the small, self-contained flake
in this directory (`templates/bootstrap/`):

| File | Role |
| --- | --- |
| `flake.nix` | A minimal flake exposing `nixosConfigurations.<HOST>`. Sets caches, trusted users and experimental features so the install works offline-ish and unsigned. |
| `configuration.nix` | A minimal but usable system: Plasma desktop, NetworkManager, Tailscale, OpenSSH, your user `adda`, a handful of CLI tools. |
| `_disko.nix` | The disk layout ([Disko](https://github.com/nix-community/disko) declarative partitioning). **You edit this per machine.** |
| `_hardware.nix` | Hardware scan result. **You regenerate this per machine.** |

The plan:

1. Boot the live ISO.
2. Describe the disks in `_disko.nix` and let Disko partition/format/mount them.
3. Generate `_hardware.nix` for this machine.
4. `nixos-install` the bootstrap flake → a minimal, bootable system.
5. Reboot, create keys/secrets, then migrate the machine into the **full**
   configuration under `modules/hosts/<HOST>/` and rebuild.

> **Conventions used below**
> - `<HOST>` — the machine's hostname, e.g. `fulgur-nixos`, `mensa-nixos`,
>   `nomen-nixos`.
> - `<USER>` — your user, i.e. `adda`.
> - `/dev/sda`, `/dev/nvme0n1` — **example** device names. Always replace with
>   the real ones from `lsblk`.

---

## 1. Download the graphical ISO

Go to the official download page: **<https://nixos.org/download/>**

Under **NixOS: the Linux distribution** pick:

> **Graphical ISO image** → **64-bit Intel/AMD** (`x86_64-linux`)

This is a hybrid live image (boots from USB, on both UEFI and legacy BIOS) that
includes the graphical Calamares installer and a full Plasma desktop — handy
because you get a browser and a terminal while installing.

### Verify the download

The download page lists a **SHA-256** next to the image. Check it:

```sh
sha256sum nixos-graphical-*-x86_64-linux.iso
```

Compare the printed hash with the one on the site. If they differ, **do not
use the image** — re-download it.

---

## 2. Write the ISO to a USB stick

Find the USB device with `lsblk` **before and after** plugging it in (so you are
certain which one it is), then write the image. This **erases the whole USB
stick**.

```sh
# Replace /dev/sdX with your USB device (NOT a partition like /dev/sdX1).
sudo dd bs=4M if=/path/to/nixos-graphical-*.iso of=/dev/sdX \
        conv=fsync oflag=direct status=progress
sync
```

> ⚠️ Double-check `of=`. Writing to the wrong device destroys its data.

---

## 3. Boot the live environment

1. Boot from the USB stick (spam the firmware boot-menu key: usually `F12`,
   `F2`, `Esc` or `Del`).
2. **Prefer UEFI.** In the boot menu choose the entry labelled
   `UEFI: <USB label>` rather than the plain/legacy one, unless the target
   machine only does legacy BIOS.
3. Wait for the Plasma live desktop, then open **Konsole**.

### Make the live session comfortable

```sh
# Become root for the whole install.
sudo -i
```

In the **Plasma live desktop** (i.e. inside Konsole), set the keyboard layout and
sleep behaviour through the GUI: **System Settings → Keyboard** for the layout,
and **System Settings → Power Management** set to never sleep — a long build
interrupted by suspend is painful.

The next two commands only make sense in a **bare TTY** (a text virtual console,
e.g. one you reach with `Ctrl+Alt+F3`, or the minimal ISO with no desktop). They
have **no effect inside Konsole/a graphical terminal**, so skip them if you're on
the Plasma desktop:

```sh
# (TTY only) set the console keyboard layout, e.g. Italian:
loadkeys it

# (TTY only) stop the text console from blanking / powersaving:
setterm -blank 0 -powersave off
```

---

## 4. Get the bootstrap files onto the live system

Clone the repository (Git is not in the live image by default, so run it via
`nix-shell`):

```sh
nix-shell -p git --run "git clone https://codeberg.org/Adda/nixos-config"
cd nixos-config/templates/bootstrap
```

You will do all the editing in this `templates/bootstrap/` directory.

---

## 5. Identify the disks

```sh
lsblk -o NAME,SIZE,TYPE,MODEL
```

Note the **whole-disk** device name(s) you want to install onto — e.g.
`/dev/sda`, `/dev/nvme0n1`. You'll put these into `_disko.nix` next.

---

## 6. Choose a disk layout and edit `_disko.nix`

Two procedures follow. **Pick one.** The unencrypted layout is the current
default; the LUKS layout is for machines that must be encrypted at rest (e.g. a
laptop that leaves the house).

Both layouts produce the same logical structure that the rest of this config
expects:

```
disk ──▶ ESP (FAT32, boot files)
     └─▶ LVM volume group "pool-ssd"
             └─▶ Btrfs
                   ├── subvolume /root  → /
                   ├── subvolume /nix   → /nix   (noatime)
                   ├── subvolume /tmp   → /tmp    (nodatacow, no snapshots)
                   └── subvolume /home  → /home
```

Why this shape:
- **LVM** gives you room to add logical volumes later without repartitioning.
- **Btrfs subvolumes** let you snapshot `/` and `/home` while keeping `/nix`
  and `/tmp` *out* of those snapshots (you never want to roll `/nix` back — Nix
  manages its own generations — and `/tmp` is disposable).
- `compress=zstd` saves space transparently; `noatime` speeds up `/nix`;
  `nodatacow` on `/tmp` avoids Btrfs copy-on-write overhead on churny writes.

---

### 6a. Default layout — **unencrypted** (LVM + Btrfs)

This is exactly what ships in [`_disko.nix`](_disko.nix). You only need to
**replace the device name** on the `device = "/dev/sda";` line with your real
disk (from step 5). The relevant part:

```nix
# _disko.nix (unencrypted — the default)
{
  disko.devices = {
    disk = {
      ssd-256gb = {
        type = "disk";
        device = "/dev/sda";              # ← TODO: your real disk
        content = {
          type = "gpt";
          partitions = {
            # Legacy BIOS boot stub (harmless on UEFI, lets the same
            # layout boot on old machines too).
            bios-boot = { size = "64M"; type = "EF02"; priority = 1; };

            # EFI System Partition.
            esp = {
              size = "2G"; type = "EF00"; priority = 2;
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot/efi";
                mountOptions = [ "umask=0077" ];
              };
            };

            # Everything else → one LVM physical volume.
            primary = {
              size = "100%";
              content = { type = "lvm_pv"; vg = "pool-ssd"; };
            };
          };
        };
      };
    };

    lvm_vg.pool-ssd = {
      type = "lvm_vg";
      lvs.root = {
        size = "100%FREE";
        content = {
          type = "btrfs";
          extraArgs = [ "-f" ];
          subvolumes = {
            "/root" = { mountpoint = "/";    mountOptions = [ "compress=zstd" "noatime" ]; };
            "/nix"  = { mountpoint = "/nix"; mountOptions = [ "compress=zstd" "noatime" ]; };
            "/tmp"  = { mountpoint = "/tmp"; mountOptions = [ "noatime" "nodatacow" ]; };
            "/home" = { mountpoint = "/home"; mountOptions = [ "compress=zstd" ]; };
          };
        };
      };
    };
  };
}
```

With this layout the shipped `configuration.nix` bootloader block (GRUB with
`efiSupport`, ESP at `/boot/efi`) works as-is. **Skip to step 7.**

> Adding a second disk / data pool? See how `mensa-nixos` chains extra disks
> into a `pool-hdd` VG in
> [`modules/hosts/mensa-nixos/_disko.nix`](../../modules/hosts/mensa-nixos/_disko.nix).

---

### 6b. Encrypted layout — **LUKS full-disk** (LUKS → LVM → Btrfs)

Same logical structure, but the whole data partition is wrapped in a **LUKS2**
container. You unlock it with a passphrase at boot; everything above it (LVM,
Btrfs, `/`, `/nix`, `/home`, swap) is encrypted at rest.

**Two things change versus the default: the disk layout and the bootloader.**

#### Why the bootloader changes

GRUB cannot reliably read **LUKS2** (cryptsetup's default format, which uses
Argon2). With GRUB you would be forced onto LUKS1 *and* two passphrase prompts
(one in GRUB, one in the initrd). The clean, modern approach is **systemd-boot**
with an **unencrypted ESP mounted at `/boot`** that holds the kernel + initrd;
the initrd then unlocks LUKS. Result: **LUKS2, a single passphrase prompt.**

Disko's module auto-generates the `boot.initrd.luks.devices.*` entry from the
`_disko.nix` below, so you do **not** need to write any initrd unlock config by
hand.

#### The encrypted `_disko.nix`

Replace the contents of `_disko.nix` with this (again, fix the `device` line):

```nix
# _disko.nix (encrypted — LUKS2 → LVM → Btrfs, systemd-boot)
{
  disko.devices = {
    disk = {
      ssd-256gb = {
        type = "disk";
        device = "/dev/sda";              # ← TODO: your real disk
        content = {
          type = "gpt";
          partitions = {
            # Unencrypted ESP. Holds the kernel + initrd + systemd-boot.
            # Mounted at /boot (NOT /boot/efi) for this layout.
            esp = {
              size = "2G"; type = "EF00";
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
                mountOptions = [ "umask=0077" ];
              };
            };

            # The encrypted container: everything else.
            luks = {
              size = "100%";
              content = {
                type = "luks";
                name = "cryptssd";        # → /dev/mapper/cryptssd
                settings = {
                  allowDiscards = true;   # let SSD TRIM through (small info leak, big lifespan/perf win)
                };
                # Used ONLY while formatting to set the initial passphrase.
                # It is NOT stored in the system; at boot you type the passphrase.
                passwordFile = "/tmp/secret.key";
                content = { type = "lvm_pv"; vg = "pool-ssd"; };
              };
            };
          };
        };
      };
    };

    # Identical VG + Btrfs subvolumes to the default layout — just sitting
    # inside the LUKS container now.
    lvm_vg.pool-ssd = {
      type = "lvm_vg";
      lvs.root = {
        size = "100%FREE";
        content = {
          type = "btrfs";
          extraArgs = [ "-f" ];
          subvolumes = {
            "/root" = { mountpoint = "/";    mountOptions = [ "compress=zstd" "noatime" ]; };
            "/nix"  = { mountpoint = "/nix"; mountOptions = [ "compress=zstd" "noatime" ]; };
            "/tmp"  = { mountpoint = "/tmp"; mountOptions = [ "noatime" "nodatacow" ]; };
            "/home" = { mountpoint = "/home"; mountOptions = [ "compress=zstd" ]; };
          };
        };
      };
    };
  };
}
```

#### Set the passphrase file *before* running Disko

`passwordFile` is read only during formatting. Create it now — **use
`-n` so there is no trailing newline in your passphrase**:

```sh
# Choose a strong passphrase; you will type it on every boot.
echo -n 'your-strong-passphrase' > /tmp/secret.key
```

#### Switch the bootloader to systemd-boot

In `configuration.nix`, **replace the entire GRUB/EFI block** with:

```nix
  # Encrypted layout: systemd-boot on an unencrypted ESP at /boot.
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.loader.efi.efiSysMountPoint = "/boot";
```

That is: delete the `boot.loader.grub.*` lines and the
`boot.loader.efi.efiSysMountPoint = "/boot/efi";` line, and add the three lines
above.

> **Must you keep GRUB?** Then use LUKS1 (`extraFormatArgs = [ "--type luks1" ]`
> in the `luks` block), keep a separate small `/boot` inside the encrypted
> volume, and set `boot.loader.grub.enableCryptodisk = true;`. Expect a
> passphrase prompt in GRUB *and* in the initrd. systemd-boot is recommended
> instead.

#### Variant: installing onto an **external USB SSD** (encrypted)

Same as the encrypted layout above, but the target is an SSD in a USB enclosure
that you want to carry between machines and boot from a USB port. Three things
change; everything else (LUKS → LVM → Btrfs, ESP at `/boot`, systemd-boot) stays
identical.

**1 — Identify the disk by a stable path, not `/dev/sdX`.** USB device letters
(`/dev/sda`, `/dev/sdb`, …) are handed out in probe order and can move between
boots — dangerous when the wrong one gets wiped. Point `_disko.nix` at the
`by-id` path, which always names the same physical disk:

```sh
ls -l /dev/disk/by-id/ | grep -i usb      # find your enclosure, e.g. usb-Samsung_Portable_SSD-0:0
```

```nix
# in _disko.nix — replace the device line
device = "/dev/disk/by-id/usb-<your-enclosure-id>";   # NOT /dev/sda
```

**2 — Make it boot on *any* machine (removable install).** For an internal disk
you let the installer register this machine's firmware boot entry
(`canTouchEfiVariables = true`). A portable disk should **not** write an entry
into every machine's NVRAM, so set it to `false`. systemd-boot still installs the
removable fallback loader (`EFI/BOOT/BOOTX64.EFI`) onto the ESP, so the disk
boots through any firmware's "boot from USB" menu:

```nix
  # configuration.nix — external/portable encrypted disk (use INSTEAD of the 6b block)
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.efiSysMountPoint = "/boot";
  boot.loader.efi.canTouchEfiVariables = false;   # ← false for a portable USB disk
```

**3 — Put USB-storage drivers in the initrd** so the encrypted root on the USB
disk is reachable before the system comes up. The hardware scan (step 8) usually
catches these because you are *already* booted from USB, but make sure the
`boot.initrd.availableKernelModules` list in `_hardware.nix` contains at least:

```nix
  boot.initrd.availableKernelModules = [ "xhci_pci" "ehci_pci" "usb_storage" "uas" "sd_mod" ];
```

> On the machine you plug it into, open the firmware boot menu and choose the USB
> disk (you may need to enable USB booting and/or disable Secure Boot first). You
> are then prompted for the LUKS passphrase exactly as with an internal disk.

---

## 7. Set the hostname

> **Where you are:** steps 7–11 all run in the **live session**, inside the
> `nixos-config/templates/bootstrap/` directory you cloned and `cd`'d into in
> step 4. Edit the files right there; the commands below read and write files
> relative to it.

Pick the host's name (e.g. `fulgur-nixos`) and set it in **two** places (both
files live in `nixos-config/templates/bootstrap/`):

- `flake.nix` → `hostName = "HOST-nixos";`
- `configuration.nix` → `networking.hostName = "HOST-nixos";`

They must match, because you install with `nixos-install --flake .#<HOST>`.

While you're in `configuration.nix`, also review:

- `time.timeZone` (default `Europe/Prague`).
- Locale / console keymap (commented out by default).
- `system.stateVersion` — leave it at the value that matches the release you
  are installing (currently `"26.05"`); **do not** bump it later.

---

## 8. Generate the hardware configuration

Run this **from `nixos-config/templates/bootstrap/`** — the `>` redirect writes
`_hardware.nix` into the current directory, next to `_disko.nix`.

This detects kernel modules, CPU microcode, etc. `--no-filesystems` is
important: Disko owns the filesystem/mount definitions, so we don't want a
second, conflicting set.

```sh
# in nixos-config/templates/bootstrap/
nixos-generate-config --no-filesystems --show-hardware-config > _hardware.nix
```

---

## 9. Partition, format and mount with Disko

Run this **from `nixos-config/templates/bootstrap/`** (the `./_disko.nix` path is
relative to it). It **destroys all data** on the target disk, creates the
partitions, formats them and mounts everything under `/mnt`.

```sh
# in nixos-config/templates/bootstrap/
nix --experimental-features "nix-command flakes" \
  run github:nix-community/disko/latest -- \
  --mode destroy,format,mount ./_disko.nix
```

**Why you can't wipe the wrong drive here:** Disko only ever touches the disks
named on the `device = "…";` lines in `_disko.nix`. `destroy,format` runs against
*those exact device paths and nothing else* — it does not scan for or reformat
any other disk, and it never touches the live USB you booted from. So once you've
confirmed each `device` line points at the intended disk (cross-check the names
against `lsblk`, or use a `/dev/disk/by-id/…` path so it can't drift), your other
drives are safe by construction.

> For the **encrypted** layout, make sure you created `/tmp/secret.key` (step
> 6b) first — Disko reads it to set the LUKS passphrase.

### Verify before installing

```sh
mount | grep /mnt          # /, /nix, /home, /boot (or /boot/efi) all present?
lsblk                      # partitions + sizes look right?
pvs; vgs; lvs              # LVM: pool-ssd / root visible?

# Encrypted layout only — confirm the mapper device is open:
cryptsetup status cryptssd
```

You should see `/mnt`, `/mnt/nix`, `/mnt/home`, `/mnt/tmp` and the ESP mounted.

---

## 10. (Optional) Swap on Btrfs

**Most of the time you do nothing here.** On the finished system the full hosts
declare swap with `addax.swapfile` (see
[`modules/swapfile.nix`](../../modules/swapfile.nix)); NixOS then creates
`/var/lib/swapfile` at activation and already disables copy-on-write for it. So
swap is automatic once you're on the full config.

**When you'd add one by hand:** the *bootstrap* flake sets up **no swap**, so
there is none during the install itself. On a **low-RAM machine**, a big step —
building the closure, or `nixos-install` — can run out of memory and get
OOM-killed. Adding a temporary swapfile in the live/target system before that
step gives the build somewhere to spill.

The extra dance below exists because **Btrfs stores a swapfile copy-on-write by
default, and the kernel refuses to `swapon` a CoW / compressed / multi-extent
file** (`swapon: Invalid argument`). You must disable CoW on the *empty* file
*before* writing any bytes, then allocate it contiguously:

```sh
truncate -s 0 /var/lib/swapfile
chattr +C /var/lib/swapfile      # disable CoW BEFORE the file has any data
fallocate -l 16G /var/lib/swapfile
chmod 600 /var/lib/swapfile
mkswap /var/lib/swapfile
swapon /var/lib/swapfile         # activate it for the current session
```

You can otherwise leave swap out of the bootstrap entirely and let the full host
config add it (`(swapfile 16)` / `(swapfile 32)`).

---

## 11. Install the bootstrap system

Still in the live session. Flakes only see files that Git tracks, so copy the
bootstrap flake into the install target, initialise a repo there, stage
everything, then install.

```sh
# Run the `cp` from nixos-config/templates/bootstrap/ — `.` is that directory.
mkdir -p /mnt/etc/nixos
cp -r . /mnt/etc/nixos/
cd /mnt/etc/nixos

# Flakes require the files to be tracked by Git.
git init -q && git add -A

# Install. The extra features match this flake's nixConfig.
nixos-install \
  --option extra-experimental-features "nix-command flakes pipe-operators" \
  --flake .#<HOST>
```

**Why you can't install onto the wrong partition:** `nixos-install` is not given
a target disk — it installs into whatever is mounted under **`/mnt`** (its
`--root` defaults to `/mnt`). Disko already mounted the correct filesystems there
in step 9, so the pieces land where those mounts point: the Nix store on
`/mnt/nix`, `/home` on `/mnt/home`, the bootloader on the ESP at `/mnt/boot` (or
`/mnt/boot/efi`), and so on. If the mount tree you checked in step 9 was right,
the install target is right — there is no separate device to get wrong.

At the end, `nixos-install` prompts for the **root password**. Set it.

> **If it fails with `error: unknown setting 'lazy-trees'` / `'submodules'`**
> (right after `Validating generated nix.conf`): those two keys in `nix.settings`
> are Lix / very-recent-Nix settings the installer's Nix doesn't recognise, and
> `nix.checkConfig` (on by default) turns that into a hard build failure. Comment
> them out in `configuration.nix` (they ship commented in the current template),
> then re-stage and re-install:
>
> ```sh
> # in /mnt/etc/nixos, edit configuration.nix so these two lines under
> # nix.settings are commented:  # lazy-trees = true;   # submodules = true;
> nano configuration.nix
> git add -A            # flakes read the (tracked) working tree — re-stage the edit
> nixos-install --option extra-experimental-features "nix-command flakes pipe-operators" --flake .#<HOST>
> ```
>
> Quick escape hatch instead of editing: add `nix.checkConfig = false;` — but
> that only silences validation and leaves the bogus settings in `nix.conf`;
> removing them is cleaner.

> On a very weak machine, don't build the whole thing locally — install just
> this bootstrap flake now, then use a remote builder for the full config
> (see step 17).

---

## 12. First boot

```sh
reboot
```

Remove the USB stick when it powers down. For the **encrypted** layout you'll be
asked for your LUKS passphrase early in boot — that's the initrd unlocking
`cryptssd`.

So far only **root** has a password — you set it at the end of `nixos-install`.
The `adda` account has **no password yet**, so you **cannot log in as `adda`** at
the Plasma login screen. You must log in as **root** first and give `adda` a
password.

The graphical login manager only lists normal users, so switch to a **text
console** to reach root: press **`Ctrl+Alt+F3`** (try `F2`–`F6` if that one is
busy), then:

```sh
# Log in as: root   (using the password you set during nixos-install)
passwd adda          # set adda's password
# passwd             # optionally change root's own password too
```

Then return to the graphical login with **`Ctrl+Alt+F1`** (or `F7`) and log in as
`adda`.

---

## 13. Bring the repo into your home and clone the full config

**So far:** you partitioned (and optionally encrypted) the disk, installed a
minimal *bootstrap* system, rebooted into it, and set passwords. You're now
logged into the new machine — the live ISO is done.

**From here on** you work from your home directory instead of `/etc/nixos`, and
you build the **full** configuration rather than the bootstrap flake. So this
step does two things: it **clones the full repo** from Codeberg into
`~/.nixos-config`, and it **rescues the two machine-specific files you generated
during install** — your edited `_disko.nix` and the scanned `_hardware.nix`.
Those live in `/etc/nixos` on this machine (that's the bootstrap flake you
installed from); the fresh clone only has the *generic* templates, so you must
carry your real ones across into the new host's directory.

```sh
# 1. Clone the FULL configuration repo (not just the bootstrap flake) into home.
nix-shell -p git --run "git clone https://codeberg.org/Adda/nixos-config ~/.nixos-config"
cd ~/.nixos-config

# 2. Rescue your generated machine files from /etc/nixos into the new host dir.
#    <HOST> = the hostname you chose in step 7 (e.g. fulgur-nixos).
mkdir -p modules/hosts/<HOST>
sudo cp /etc/nixos/_disko.nix    modules/hosts/<HOST>/_disko.nix
sudo cp /etc/nixos/_hardware.nix modules/hosts/<HOST>/_hardware.nix

# 3. Take ownership (the clone is already yours; the /etc/nixos copies come in as root).
sudo chown -R "$USER" ~/.nixos-config
```

> **`git clone` needs the target to be empty or non-existent.** If a previous
> attempt left a partial `~/.nixos-config`, step 1 fails with *"destination path
> '…' already exists and is not an empty directory"*. Clear it first —
> `rm -rf ~/.nixos-config` — then re-run, or clone to a temporary path and move it
> into place.

Your machine-specific `_disko.nix` / `_hardware.nix` now sit in
`modules/hosts/<HOST>/`, ready for step 16. The repo's `templates/bootstrap/`
files stay pristine (they're the generic templates, not your machine's).

---

## 14. Generate keys and secrets

### SSH key (Ed25519)

Create it **without a passphrase first** (some later steps read the raw key),
then add a passphrase afterwards.

```sh
ssh-keygen -t ed25519 -a 100 -C "adda@<HOST>"     # press Enter twice: no passphrase yet
```

### GPG key (for signing commits)

```sh
gpg --full-generate-key
#   → (9) ECC (sign and encrypt)  [default]
#   → (1) Curve 25519             [default]
#   → 0 = key does not expire (or choose an expiry)

# Find the key id (the long id after "sec"):
gpg --list-secret-keys --keyid-format LONG

# Export the public key if you need to add it to Codeberg/GitHub:
gpg --armor --export <GPG_KEY_ID>
```

### SOPS / age keys (secrets decryption)

This config uses [sops-nix](https://github.com/Mic92/sops-nix) with **age keys
derived from SSH** (see
[`modules/aspects/security/sops.nix`](../../modules/aspects/security/sops.nix),
which uses `/etc/ssh/ssh_host_ed25519_key`).

```sh
mkdir --parents "$HOME/.config/sops/age/"

# Derive your user age key from your SSH key:
nix-shell --packages ssh-to-age --run \
  "ssh-to-age -private-key -i $HOME/.ssh/id_ed25519 > $HOME/.config/sops/age/keys.txt"

# Now that the key is derived, add a passphrase to the SSH key:
ssh-keygen -p -f "$HOME/.ssh/id_ed25519"

# Print the PUBLIC age keys to register in .sops.yaml:
nix-shell --packages age --run "age-keygen -y $HOME/.config/sops/age/keys.txt"   # user key  → adda-<HOST>
nix-shell --packages ssh-to-age --run "ssh-to-age -i /etc/ssh/ssh_host_ed25519_key.pub"  # host key → host-<HOST>
```

Add **both** public keys to [`.sops.yaml`](../../.sops.yaml) following the
existing pattern, under new anchors `&adda-<HOST>` and `&host-<HOST>`, and list
them in the `key_groups.age` block:

```yaml
keys:
  - &adda-<HOST> age1...     # from age-keygen -y
  - &host-<HOST> age1...     # from ssh-to-age of the host pubkey
creation_rules:
  - path_regex: secrets/[^/]+\.(yaml|json|env|ini)$
    key_groups:
      - age:
          # ...existing entries...
          - *adda-<HOST>
          - *host-<HOST>
```

### Re-encrypt the secrets for the new keys

Two gotchas here on a fresh machine:

- **`sops` isn't installed yet.** The bootstrap system doesn't ship it — it's
  the full config (`modules/aspects/security/sops.nix`) that adds `sops`, `age`
  and `ssh-to-age`, and that only takes effect after step 16. So run `sops` via
  `nix-shell`, **not** the `just sops-update` recipe (which calls a bare `sops`
  and fails with `Command 'sops' not found`). The `just` shortcut works once
  you're on the full system.
- **`updatekeys` must decrypt the file first**, using a key that is *already* a
  current recipient. This machine's freshly generated keys are **not** yet
  recipients, so the rekey has to run on an **existing trusted machine**
  (`mensa-nixos`, `nomen-nixos`, …) that can already decrypt — then you `git
  pull` the re-encrypted file here.

```sh
# On a machine that can already decrypt the secrets (after adding this host's
# new age keys to .sops.yaml and committing):
nix-shell -p sops --run "sops updatekeys --yes secrets/secrets.yaml"

# Later, once THIS machine is on the full config (step 16), the shortcut works:
#   just sops-update
```

If this is your only machine and nothing can decrypt the existing secret yet,
start it over from scratch instead (you lose the old value):

```sh
rm -f ~/.nixos-config/secrets/*
grep -r "sops.secrets" .    # see which secrets the config expects
EDITOR=hx nix-shell --packages sops --run "sops secrets/secrets.yaml"
```

---

## 15. Set your Git identity

Edit [`modules/aspects/console/vcs/git.nix`](../../modules/aspects/console/vcs/git.nix)
and set, under `programs.git`:

- `settings.user.name` and `settings.user.email` (currently `Spiraliton` /
  `alexramonda@gmail.com`).
- `signing.key` — replace with the GPG key id from step 14
  (currently `70C33476D3701ECA`). `signByDefault = true` means every commit is
  signed, so this must be a key that exists on the machine.

---

## 16. Register the host and switch to the full configuration

1. Confirm your host directory already holds the two machine files you rescued in
   step 13:

   ```sh
   ls modules/hosts/<HOST>/     # → _disko.nix  _hardware.nix
   ```

   If they're missing (you skipped that part of step 13), copy them now from
   `/etc/nixos` — **not** from `templates/bootstrap/`, whose copies are the
   generic templates, not your machine's:

   ```sh
   mkdir -p modules/hosts/<HOST>
   sudo cp /etc/nixos/_disko.nix    modules/hosts/<HOST>/_disko.nix
   sudo cp /etc/nixos/_hardware.nix modules/hosts/<HOST>/_hardware.nix
   sudo chown "$USER" modules/hosts/<HOST>/_disko.nix modules/hosts/<HOST>/_hardware.nix
   ```

2. Add a `system.nix` for the host, modelled on an existing one such as
   [`modules/hosts/fulgur-nixos/system.nix`](../../modules/hosts/fulgur-nixos/system.nix).
   The key parts: define the host + user, and **import Disko + your files**:

   ```nix
   den.aspects.<HOST> = {
     nixos.imports = [
       inputs.disko.nixosModules.disko   # pull in the disko module
       ./_disko.nix
       ./_hardware.nix
     ];
     includes = with addax; [ (swapfile 16) ];   # optional swap, games, etc.
   };
   ```

   > **Encrypted (LUKS2) host?** You need a few extra edits on top of the
   > generic flow above — see **16b** immediately below before rebuilding.

3. Stage the new files (flakes ignore untracked files) and rebuild:

   ```sh
   git add -A
   just switch          # → nh os switch --impure   (or: sudo nixos-rebuild switch --flake .#<HOST>)
   ```

   Other handy recipes (see [`justfile`](../../justfile)): `just boot`,
   `just build`, `just check`, `just update`, `just system-update`.

### 16b. Extra steps for a LUKS2-encrypted host

If you installed with the **encrypted** layout (step 6b), the host needs three
changes on top of the generic flow above. The reason: the full configuration
**force-enables legacy-BIOS GRUB for every host** in
[`modules/defaults.nix`](../../modules/defaults.nix), and GRUB cannot read LUKS2.
So the encrypted host must (1) carry the LUKS `_disko.nix`, (2) be *allowed* to
pick its own bootloader, and (3) actually switch to systemd-boot.

**1 — Use the encrypted `_disko.nix`.** The file you copied to
`modules/hosts/<HOST>/_disko.nix` must be the **LUKS2 version** from step 6b
(ESP at `/boot`, `luks` → `lvm_pv` → `pool-ssd` → Btrfs), *not* the default
unencrypted one. Disko's module (already imported) auto-generates the
`boot.initrd.luks.devices.cryptssd` entry, so the initrd unlocks the disk at
boot with no extra config from you.

**2 — Let hosts choose their own bootloader.** In
[`modules/defaults.nix`](../../modules/defaults.nix) the bootloader is *forced*:

```nix
# modules/defaults.nix — BEFORE
boot.loader.systemd-boot.enable      = lib.mkForce false;
boot.loader.grub.enable              = lib.mkForce true;
boot.loader.efi.canTouchEfiVariables = false;
```

`lib.mkForce` beats any per-host value, so no host can opt into systemd-boot
while these are forced. Relax them to `lib.mkDefault` — GRUB stays the default
for the other hosts, but a host may now override it:

```nix
# modules/defaults.nix — AFTER
boot.loader.systemd-boot.enable      = lib.mkDefault false;
boot.loader.grub.enable              = lib.mkDefault true;
boot.loader.efi.canTouchEfiVariables = lib.mkDefault false;
```

This is safe: `fulgur-nixos`, `mensa-nixos` and `nomen-nixos` set no bootloader
of their own, so they keep legacy-BIOS GRUB exactly as before, and nothing else
in the repo enables systemd-boot.

**3 — Switch the encrypted host to systemd-boot.** Add a small module
`modules/hosts/<HOST>/_boot.nix`:

```nix
# modules/hosts/<HOST>/_boot.nix
{
  boot.loader = {
    grub.enable             = false;
    systemd-boot.enable     = true;
    efi.canTouchEfiVariables = true;
    efi.efiSysMountPoint     = "/boot";   # must match the ESP mountpoint in _disko.nix
  };
}
```

and import it from the host, so its `nixos.imports` reads:

```nix
den.aspects.<HOST>.nixos.imports = [
  inputs.disko.nixosModules.disko
  ./_disko.nix
  ./_hardware.nix
  ./_boot.nix          # ← encrypted host only
];
```

Then stage (`git add -A`) and rebuild (`just switch`) as in step 16.3. On the
next boot you'll be prompted **once** for the LUKS passphrase (in the initrd),
after which systemd-boot loads the kernel from the unencrypted ESP.

> **Encrypting your *current* machine (`fulgur-nixos`)?** You don't need a new
> host — apply steps 1–3 above to `fulgur-nixos` directly (swap its `_disko.nix`
> for the LUKS version, relax `defaults.nix`, add `_boot.nix`). Create a
> brand-new host directory only when it's a genuinely *different* machine: a host
> bundles machine-specific pieces — the `_hardware.nix` scan, the disk device in
> `_disko.nix`, the hostname, the `nixos-hardware` profile (e.g.
> `lenovo-thinkpad-t14-amd-gen4`) and the swap size — so pointing `fulgur` at
> another laptop would overwrite it.

> **Back up your LUKS header and keep a recovery key** before trusting the
> machine — see [Appendix C](#appendix-c--luks-safety-header-backup--recovery).

#### External USB SSD host — two more things

The external-disk variant (step 6b → "external USB SSD") needs a bit more here,
because two of its settings lived in the bootstrap `configuration.nix` /
`_hardware.nix`, and you leave both of those behind when you switch to the full
config. Fold them into the host's `_boot.nix` so they persist:

```nix
# modules/hosts/<HOST>/_boot.nix  (external USB SSD variant)
{
  boot.loader = {
    grub.enable              = false;
    systemd-boot.enable      = true;
    efi.efiSysMountPoint     = "/boot";
    efi.canTouchEfiVariables = false;   # ← false (not true): keep it portable/removable
  };

  # Keep USB-storage drivers in the initrd HERE, durably — `_hardware.nix` is
  # auto-generated and a later `nixos-generate-config` would drop manual edits.
  boot.initrd.availableKernelModules = [ "xhci_pci" "ehci_pci" "usb_storage" "uas" "sd_mod" ];
}
```

- `canTouchEfiVariables = false` is the one real difference from the internal
  `_boot.nix` above (`true`). Without it, each machine you plug into gets a boot
  entry written to its firmware; with it, systemd-boot's removable fallback
  (`EFI/BOOT/BOOTX64.EFI`) keeps the disk bootable anywhere via the USB boot menu.
- The `availableKernelModules` list merges with the one in `_hardware.nix` (it's
  an additive list), so having it in both is fine and guarantees the USB drivers
  are always in the initrd.

> **Booting it on more than one machine?** Then in `system.nix` do **not** import
> a machine-specific `nixos-hardware` profile (e.g. `lenovo-thinkpad-t14-amd-gen4`)
> — it would apply one laptop's quirks everywhere. Also note `_hardware.nix` is
> tailored to the machine you generated it on (CPU microcode, `kvm-intel` vs
> `kvm-amd`, GPU modules); for a truly cross-machine stick you may want to
> broaden it or drop the CPU/GPU-specific bits. For a single-machine external
> boot, the normal host setup is fine as-is.

---

## 17. (Optional) Use a remote builder for weak machines

If the target is too slow to build the full closure, build it elsewhere and copy
it over. On the target, allow unsigned closures and trust yourself:

```nix
nix.settings.require-sigs = false;
nix.settings.trusted-users = [ "adda" "@wheel" "root" ];
```

Bring the target onto your tailnet, then build remotely:

```sh
# On the target:
sudo tailscale up --ssh

# On the builder:
nix build .#nixosConfigurations.<HOST>.config.system.build.toplevel
nix copy --to ssh://<HOST> ./result      # or: nix-copy-closure --to <HOST> ./result

# Back on the target:
sudo nixos-rebuild boot --flake .#<HOST>

# …or drive it all from the builder in one go:
nixos-rebuild boot --flake .#<HOST> --target-host <HOST> --sudo --ask-sudo-password
```

---

## Appendix A — `nixos-enter` to repair an installed Btrfs system

**When you need this:** the machine is already installed but won't boot or needs
fixing from outside — a broken bootloader, a bad configuration that fails to
activate, a forgotten root password, or a file you need to edit before it will
come up. You boot the live ISO again and `nixos-enter` chroots you *into* the
on-disk system so you can run commands as if you had booted it normally. The
mounts have to be recreated by hand first, because the live ISO knows nothing
about your Disko layout (and, for the encrypted layout, the disk is still locked).

If you need to chroot back into the installed system from the live ISO:

```sh
sudo -i
mount -t btrfs -o subvol=root /dev/pool-ssd/root /mnt
mkdir -p /mnt/{home,nix,tmp,boot}
mount -t btrfs -o subvol=home /dev/pool-ssd/root /mnt/home
mount -t btrfs -o subvol=nix  /dev/pool-ssd/root /mnt/nix
mount -t btrfs -o subvol=tmp  /dev/pool-ssd/root /mnt/tmp
mount /dev/disk/by-partlabel/disk-ssd-256gb-esp /mnt/boot   # or /mnt/boot/efi
# Encrypted: cryptsetup open /dev/sdaN cryptssd  first, then mount /dev/pool-ssd/root
nixos-enter
```

---

## Appendix B — Post-install odds and ends

Collected from real installs (`notREADME.md`):

- **Nushell/carapace error on first terminal launch:**

  ```sh
  mkdir -p ~/.cache/nushell
  carapace _carapace nushell | save --force $"($nu.cache-dir)/carapace.nu"
  ```

- **Plasma keyboard layout:** search for `kxkbrc` / `Layout` in the desktop
  aspect (`modules/aspects/desktop/`).
- **xremap** key remapping: check `modules/aspects/*xremap*` for correct
  device/usage.
- **Legacy-BIOS GRUB** not installing? From inside `nixos-enter`:
  `grub-install --target=i386-pc /dev/sda`, then `exit`, `umount -lR /mnt`,
  `reboot`.
- If a build breaks on a specific package during bootstrap (e.g. `bottles`) or a
  custom kernel, temporarily comment it out / switch to the stock kernel to get
  a bootable system, then re-enable once you're on the full config.
- **Log in to services** afterwards, e.g. `atuin login`.

---

## Appendix C — LUKS safety: header backup & recovery

*(Encrypted layout only.)* A LUKS volume becomes **unrecoverable if its header
is damaged** — a bad write to the start of the disk, a botched partition edit —
even when you know the passphrase. Do this once, soon after install.

**Back up the header** to somewhere *off* the encrypted disk (a USB stick,
another machine, a password manager):

```sh
# <LUKS_PART> = the RAW partition, e.g. /dev/sda2 or /dev/nvme0n1p2 — NOT /dev/mapper/cryptssd.
sudo cryptsetup luksHeaderBackup /dev/<LUKS_PART> \
  --header-backup-file luks-header-<HOST>.img
```

Keep it safe: anyone with the header **and** your passphrase can open the disk.
Restore with `cryptsetup luksHeaderRestore /dev/<LUKS_PART> --header-backup-file
luks-header-<HOST>.img`.

**Add a second key slot** so one forgotten or mistyped passphrase can't lock you
out (LUKS has up to 8 slots):

```sh
sudo cryptsetup luksAddKey /dev/<LUKS_PART>   # asks for an existing passphrase, then the new one
sudo cryptsetup luksDump   /dev/<LUKS_PART>   # shows which key slots are in use
```

> Want this automatic at format time instead? Disko's `luks` type accepts
> `enrollRecovery = true;` (generates a high-entropy recovery passphrase and
> shows a QR code) and `enrollFido2 = true;` (unlock with a hardware security
> key). Add either to the `luks` block in `_disko.nix` *before* running Disko.

**Periodic TRIM.** The encrypted layout sets `allowDiscards = true`, letting
TRIM pass through LUKS to the SSD (a small confidentiality trade-off for real
lifespan/performance gains). Enable the weekly trim timer on the host:

```nix
services.fstrim.enable = true;
```

---

## Quick reference

| Step | Command |
| --- | --- |
| Write USB | `sudo dd bs=4M if=nixos-*.iso of=/dev/sdX conv=fsync oflag=direct status=progress` |
| Hardware scan | `nixos-generate-config --no-filesystems --show-hardware-config > _hardware.nix` |
| Partition (Disko) | `nix run github:nix-community/disko/latest -- --mode destroy,format,mount ./_disko.nix` |
| Install | `nixos-install --option extra-experimental-features "nix-command flakes pipe-operators" --flake .#<HOST>` |
| Rekey secrets | `just sops-update` |
| Rebuild | `just switch` |

Sources for the external facts in this guide: the
[NixOS download page](https://nixos.org/download/), the
[Disko examples](https://github.com/nix-community/disko/tree/master/example),
and the [NixOS Full Disk Encryption wiki](https://wiki.nixos.org/wiki/Full_Disk_Encryption).
