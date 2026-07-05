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

# (Optional) set your console keyboard layout, e.g. Italian:
loadkeys it

# Stop the screen from blanking / the machine from sleeping mid-install:
setterm -blank 0 -powersave off
```

Also open **System Settings → Power Management** in the live desktop and set it
to never sleep — a long build interrupted by suspend is painful.

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

---

## 7. Set the hostname

Pick the host's name (e.g. `fulgur-nixos`) and set it in **two** places:

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

This detects kernel modules, CPU microcode, etc. `--no-filesystems` is
important: Disko owns the filesystem/mount definitions, so we don't want a
second, conflicting set.

```sh
nixos-generate-config --no-filesystems --show-hardware-config > _hardware.nix
```

---

## 9. Partition, format and mount with Disko

This **destroys all data** on the target disk, creates the partitions, formats
them and mounts everything under `/mnt`.

```sh
nix --experimental-features "nix-command flakes" \
  run github:nix-community/disko/latest -- \
  --mode destroy,format,mount ./_disko.nix
```

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

The full hosts add swap via `addax.swapfile` (see
[`modules/swapfile.nix`](../../modules/swapfile.nix)), which creates
`/var/lib/swapfile`. On **Btrfs** a swapfile must have **copy-on-write
disabled** or the kernel refuses it. NixOS handles this for
`swapDevices`-managed swapfiles, but if you ever create one by hand:

```sh
truncate -s 0 /var/lib/swapfile
chattr +C /var/lib/swapfile      # disable CoW BEFORE writing data
fallocate -l 16G /var/lib/swapfile
chmod 600 /var/lib/swapfile
mkswap /var/lib/swapfile
```

You can leave swap out of the bootstrap and add it when you move to the full
host config (`(swapfile 16)` / `(swapfile 32)`).

---

## 11. Install the bootstrap system

Flakes only see files that Git tracks, so initialise a repo in the install
target and stage everything, then install.

```sh
# Copy the bootstrap flake into the installer target.
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

At the end, `nixos-install` prompts for the **root password**. Set it.

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

Log in as `adda` (or root) and set passwords if you didn't already:

```sh
passwd root
passwd adda
```

---

## 13. Bring the repo into your home and clone the full config

```sh
# Copy the installed config out of /etc/nixos and take ownership.
cp -r -a /etc/nixos/* ~/.nixos-config/ 2>/dev/null || \
  nix-shell -p git --run "git clone https://codeberg.org/Adda/nixos-config ~/.nixos-config"
sudo chown -R "$USER" ~/.nixos-config
cd ~/.nixos-config
```

Keep the machine-specific `_disko.nix` and `_hardware.nix` you just made — you
will drop them into the new host directory in step 16.

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

From a machine (or this one) that can already decrypt the secrets, rekey them so
the new host's keys are included:

```sh
just sops-update           # → sops updatekeys --yes secrets/*
```

If you're starting the secrets from scratch instead:

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

1. Create `modules/hosts/<HOST>/` and copy your two machine files into it:

   ```sh
   mkdir -p modules/hosts/<HOST>
   cp ~/.nixos-config/templates/bootstrap/_disko.nix    modules/hosts/<HOST>/_disko.nix
   cp ~/.nixos-config/templates/bootstrap/_hardware.nix modules/hosts/<HOST>/_hardware.nix
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
