# Bare-Metal Malware Analysis Lab — Build & Recovery Runbook

**Machine:** Lenovo Yoga — Intel Core Ultra 7, 16 GB RAM, 1 TB NVMe SSD
**Goal:** A dual-boot, bare-metal malware-analysis workstation with full-disk restore points.

| Side | OS | Tooling |
|------|----|---------|
| Windows | Windows 11 (local account only) | **FLARE-VM** + **Rusty Toolz** |
| Linux | Ubuntu 24.04 Desktop | **REMnux** + **Rusty Toolz** |
| Backup | 2 TB external SSD (Ventoy, NTFS) | **Clonezilla** full-disk images |

> These are personal research notes written *after* a real (messy) build. The
> "Right Way" sections are the clean path; the **⚠️ Pitfall** callouts are the
> exact things that bit us and how the clean path avoids them. Read the
> **Golden Rules** first — most of the pain came from breaking them.

---

## Golden Rules (read these first)

1. **Windows first, then Linux.** Installing Linux first, then Windows, lets the
   Windows installer clobber the bootloader. Windows → Ubuntu means GRUB ends up
   in control of a working dual-boot.
2. **Boot Clonezilla from a SEPARATE USB stick — never from the Ventoy SSD you're
   imaging to.** Booting Clonezilla off the Ventoy drive locks that partition
   (loopback serving the ISO), and you physically cannot mount it as the image
   target. This was the single biggest time-sink. One dedicated Clonezilla stick
   solves it permanently.
3. **Never physically unplug the Ventoy SSD while any OS is running.** Yanking it
   live sets the NTFS "dirty" flag and corrupts in-flight images. Always
   clean-eject, and keep it plugged through reboots.
4. **Local account only on the Windows side.** No Microsoft account on a box that
   will run live malware.
5. **Image before you ever open a real sample.** A restore point is only "golden"
   if it's pristine.

---

## Materials / Prep

- **Internal:** the machine's 1 TB NVMe (becomes `nvme0n1` under Linux).
- **Ventoy SSD (2 TB external):** holds ISOs *and* the Clonezilla images.
  Ventoy data partition is NTFS, labeled `Ventoy` (`sda1`); `VTOYEFI` is `sda2`.
- **A separate small USB stick (≥ 1 GB):** dedicated **Clonezilla boot** stick.
- **ISOs** (put the OS installers on the Ventoy drive; Clonezilla goes on the
  separate stick — see Rule 2):
  - Windows 11
  - Ubuntu 24.04 (Desktop — gives you a GUI out of the box; see Phase 4 note)
  - Clonezilla Live (`clonezilla-live-*-amd64.iso`)
- **Optional but recommended:** a USB-Ethernet adapter (see the FLARE Wi-Fi
  pitfall in Phase 5).

### Make the dedicated Clonezilla boot stick (do this once)

Simplest — write the ISO straight to the spare stick with **Rufus**:

1. Rufus → **Device** = the spare stick → **Boot selection** = the Clonezilla ISO
   → **START** → choose **ISO mode** if prompted.
2. That's your permanent Clonezilla boot stick. Keep it — you'll reuse it for
   every save *and* restore.

*(Alternatively: install Ventoy on the spare stick and just copy the Clonezilla
`.iso` onto it — Ventoy boots any ISO you drop on. Either way, the point is it's
a **different physical stick** from the 2 TB target.)*

---

## The Right-Way Build Sequence (overview)

0. Prep drives + build the separate Clonezilla stick.
1. Install **Windows 11** (local account).
2. **Pre-create the Linux partition in Windows Disk Management.**
3. Install **Ubuntu 24.04** into that partition.
4. Confirm **GRUB** dual-boot works.
5. *(Optional)* Clonezilla **baseline** image of the clean dual-boot.
6. Install **REMnux** (Ubuntu).
7. Set **GRUB default = Windows** (so FLARE's reboots resume).
8. Install **FLARE-VM** (Windows).
9. Install **Rusty Toolz** (both sides).
10. Clonezilla **golden** image (fully tooled).
11. **Restore** whenever the box gets contaminated.

---

## Phase 1 — Windows 11

1. Boot the Windows 11 installer (from the Ventoy menu).
2. Install to the internal NVMe. Let it create its default partitions (ESP, MSR,
   Windows C:, WinRE recovery).
3. **Force a local account** — do not sign into a Microsoft account:
   - At the "let's connect you to a network" / sign-in step, open a command
     prompt with **Shift+F10** and run:
     ```
     start ms-cxh:localonly
     ```
   - Create the local user, finish setup.
4. Boot to the desktop.

> **⚠️ Pitfall — "You've chosen Windows 11 Home" with no edition picker.** If you
> never got a version-selection screen, the ISO/key silently picked an edition.
> Home is fine for this lab, but note: on **Home**, Defender's Real-Time
> Protection re-enables itself on reboot by design (see Phase 5).

---

## Phase 2 — Pre-create the Linux partition (in Windows)

**Do this from Windows *before* booting the Ubuntu installer.** It is the single
fix for the Ubuntu-installer-can't-see-free-space problem.

1. Shrink the Windows volume to free space for Linux (Disk Management →
   right-click C: → **Shrink Volume**), *or* if you already have unallocated
   space, use it.
2. On the unallocated space → **New Simple Volume** → accept defaults (any
   filesystem/letter; Ubuntu will reformat it). This gives it a real partition
   entry.

> **⚠️ Pitfall — Ubuntu's installer (Subiquity) couldn't see the free space.**
> With raw *unallocated* space, the Ubuntu Server installer showed no way to add
> a partition (no "Add GPT Partition"; `sgdisk -e` didn't help). Creating an
> actual **partition/volume in Windows Disk Management first** makes the Ubuntu
> installer show it as a normal partition you can then reformat. This is why we
> pre-create it here rather than in the Linux installer.

---

## Phase 3 — Ubuntu 24.04

1. Boot the Ubuntu installer (Ventoy).
2. At the storage step, choose **manual / "Something else"** partitioning.
3. Find the partition you made in Phase 2 → **Edit** it:
   - Format: **ext4**
   - Mount point: **`/`**
4. The existing EFI System Partition (the Windows ESP, `nvme0n1p1`) is
   auto-selected for **`/boot/efi`** — leave it.
5. Finish the install. Create your user (or add one after — see below).

**First boot:**

> **⚠️ Pitfall — "stuck at cloud-init".** On first boot Ubuntu can sit on a
> cloud-init line. It's normal — wait, or press **Enter**. To stop it recurring:
> ```bash
> sudo touch /etc/cloud/cloud-init.disabled
> ```

**Create / fix your user if needed:**
```bash
sudo adduser <name>
sudo usermod -aG sudo <name>
# remove an unwanted user later (run as a *different* sudo user):
sudo deluser --remove-home <olduser>
```

---

## Phase 4 — Confirm the dual-boot (GRUB)

1. Reboot. You should land at **GRUB** showing **Ubuntu** and **Windows Boot
   Manager**.
2. Boot each once to confirm both work.

Reference — the final disk layout (`lsblk -f`) looks like:

```
nvme0n1                       # internal 1 TB (image THIS)
├─nvme0n1p1  vfat   FAT32     /boot/efi   (ESP: GRUB + Windows Boot Mgr)
├─nvme0n1p2                   (Microsoft reserved / MSR)
├─nvme0n1p3  ntfs             (Windows C:)
├─nvme0n1p4  ntfs             (Windows recovery)
└─nvme0n1p5  ext4             /           (Ubuntu root)
sda                           # 2 TB Ventoy external (image TARGET)
├─sda1       ntfs   Ventoy    /media/.../Ventoy   (ISOs + Clonezilla images)
└─sda2       vfat   VTOYEFI
```

> **Ubuntu Desktop vs Server:** Desktop gives you a **GNOME GUI** immediately,
> which you want for the graphical REMnux tools (Ghidra, Wireshark, PDF viewers).
> If you installed **Server** (no GUI), add one after REMnux:
> ```bash
> sudo apt install -y ubuntu-desktop-minimal
> sudo systemctl set-default graphical.target
> sudo reboot
> ```
> (`xfce4` is a lighter alternative. Revert to headless with
> `sudo systemctl set-default multi-user.target`.)

---

## Phase 5 — REMnux (Ubuntu)

Run as your normal user. Needs internet; the install pulls a *lot* (~1 hr).

```bash
ping -c 3 8.8.8.8                       # confirm the box is online first
curl -O https://REMnux.org/remnux
sha256sum remnux                        # verify against docs.remnux.org
chmod +x remnux
sudo mv remnux /usr/local/bin
sudo remnux install                     # ~1 hour; holds the terminal
```

**How you know it's done:** the `remnux install` command **returns to the shell
prompt** and prints a SaltStack summary ending in `Failed: 0`. It will look
frozen for long stretches while downloading — that's normal; do **not** Ctrl-C.

**Verify:**
```bash
remnux version
which capa oletools yara wireshark
```

---

## Phase 6 — Set GRUB default to Windows (before FLARE)

FLARE-VM reboots ~a dozen times via Boxstarter and must land back in **Windows**
each time to auto-resume. GRUB defaults to Ubuntu, which stalls the install.
Point GRUB at Windows for the duration (you can leave it this way permanently if
you like — the menu still lets you pick Ubuntu each boot).

From **Ubuntu**:
```bash
# 1. get the EXACT Windows menu-entry title
awk -F\' '/menuentry / {print $2}' /boot/grub/grub.cfg
#    -> e.g.  Windows Boot Manager (on /dev/nvme0n1p1)

# 2. edit the default
sudo nano /etc/default/grub
#    set:
#      GRUB_DEFAULT="Windows Boot Manager (on /dev/nvme0n1p1)"
#      GRUB_TIMEOUT=5

# 3. apply + reboot into Windows
sudo update-grub
sudo reboot
```

> **Rock-solid alternative** (immune to entry renames): set
> `GRUB_DEFAULT=saved` + `GRUB_SAVEDEFAULT=true`, `sudo update-grub` — GRUB then
> always boots whatever you picked last. To put Ubuntu back as default later:
> `GRUB_DEFAULT=0` → `sudo update-grub`.

---

## Phase 7 — FLARE-VM (Windows)

> **⚠️ This box has no VM snapshots — the Clonezilla image IS your snapshot.**
> The FLARE installer will warn "run this in a VM / take a snapshot." That's what
> the Phase 10 golden image is for. (Ideally capture a baseline first — Phase 8.)

**1. Disable Defender — order matters:**
- Windows Security → Virus & threat protection → **Manage settings**:
  - **Tamper Protection → OFF**
  - **Real-time protection → OFF**
  - Cloud-delivered protection + automatic sample submission → OFF
- Then, in an **elevated PowerShell** (RTP already off so AMSI doesn't block it):
  ```powershell
  Add-MpPreference -ExclusionPath 'C:\'
  ```

> On **Windows Home**, RTP re-enables itself on reboot by design. The `C:\`
> exclusion (plus, if needed, a Defender policy registry key) is what keeps FLARE
> tools from being quarantined across the reboots.

**2. Get and run FLARE-VM:**
```powershell
# download flare-vm-main.zip from github.com/mandiant/flare-vm, extract to C:\flare-vm
cd C:\flare-vm
Unblock-File .\install.ps1
Set-ExecutionPolicy Bypass -Scope LocalMachine -Force   # or: Unrestricted (FLARE's documented value)
.\install.ps1
```
- The `Unblock-File` step strips the "downloaded from the internet" mark so the
  script won't prompt. `Bypass` vs `Unrestricted`: identical here once unblocked;
  execution policy isn't a security boundary, so machine-wide `Bypass` is fine on
  a disposable lab box.
- Confirm the config prompt, then leave it on **power + network**. Boxstarter
  auto-logs-in and resumes through each reboot (that's why Phase 6 matters).

> **⚠️ Pitfall — Wi-Fi reboot race fails packages (e.g. Ghidra).** After an
> auto-login reboot, Boxstarter resumes *before* Wi-Fi reconnects (a few-second
> gap), so the next package download fails. It's not corruption.
> - **Fix:** re-run `.\install.ps1` — it's idempotent, skips what's installed,
>   retries the rest. Repeat until a run reports **zero failures**. Check
>   `C:\ProgramData\chocolatey\logs\chocolatey.log` for failures.
> - **Durable fix:** use **wired Ethernet** — it links at boot with no delay, so
>   the race disappears.

**3. Verify before imaging:** launch **IDA**, **Ghidra**, **x32dbg/x64dbg** — and
confirm the FLARE-VM desktop/wallpaper is set. Only a **zero-failure** run + tools
launching = truly done.

---

## Phase 8 — Clonezilla images (the payoff)

> **Rule 2 lives here.** Boot Clonezilla from the **separate stick**, keep the
> **2 TB Ventoy SSD as target only**. Booting off the Ventoy drive = guaranteed
> "Device or resource busy" and you cannot save. See the troubleshooting table.

Capture **two** images (different names — keep both):

| Name | When | Purpose |
|------|------|---------|
| `baseline-dualboot-<date>` | clean dual-boot (+ optionally REMnux), pre-FLARE | deep fallback |
| `golden-dualboot-tooled-<date>` | FLARE + REMnux + Rusty Toolz all verified | everyday "reset the box" |

### Save procedure (`savedisk`)

1. **Clean-shutdown** the running OS. Plug in **both** the Clonezilla stick and
   the Ventoy SSD.
2. Power on → **one-time boot menu** (Lenovo: **F12** / Fn+F12) → pick the
   **Clonezilla USB stick** (bypasses GRUB).
3. Take the **default** Clonezilla live entry → English → *Don't touch keymap* →
   **Start Clonezilla**.
4. **device-image → local_dev** → wait ~5 s → **Ctrl-C** → pick the **2 TB Ventoy
   partition** (`sda1`, ntfs, ~2T — *not* the internal nvme).
5. Directory → **`/` (top)** → Tab to **Done** (Clonezilla creates the image
   folder itself; don't descend into subfolders).
6. **Beginner** mode → **savedisk**.
7. **Name** it (`baseline-…` or `golden-…`, a new name each time).
8. Source disk → **`nvme0n1`** (the 1 TB internal — *not* the 2 TB `sda`).
9. Accept defaults → confirm **y** (and **y** again if asked) → it images
   (~15–40 min). **Don't unplug; keep on power.**
10. **Success screen** → **poweroff** → clean-eject both drives.

### Verify an image (optional, recommended for a backup you rely on)
Clonezilla → device-image → local_dev → `sda1` → `/` → **`chk-img-restorable`**.
Read-only; confirms the image is intact without touching your disk.

---

## Phase 9 — Restore (`restoredisk`)

Same setup as save — **boot the separate stick, Ventoy SSD as source.**

1. Boot the Clonezilla stick → Start Clonezilla → **device-image → local_dev** →
   pick **`sda1`** → directory **`/`**.
2. **Beginner** → **`restoredisk`**.
3. Choose the image (e.g. `golden-dualboot-tooled-<date>`).
4. Target disk → **`nvme0n1`**.
5. Confirm **twice** (`y`, `y`). ⚠️ **This overwrites the entire internal disk** —
   partition table, ESP, GRUB, both OSes. Anything on the box since the image is
   gone (that's the point: reset a contaminated lab to clean).
6. Reboot → the full dual-boot comes back exactly as imaged.

Recovery time: ~15–30 min. Restore the **golden** image for a ready-to-work lab;
the **baseline** is the deeper fallback.

---

## Rusty Toolz install (both sides)

Repo: `github.com/denvercoder/rusty-tools`. Dashboard binds **localhost only**.

**Windows:**
```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
# dashboard at http://localhost
```

**Linux:**
```bash
chmod +x install.sh      # exec bit is lost when the script is committed from Windows
./install.sh
```

> **⚠️ Pitfall — `install.sh` "Permission denied".** The script loses its Unix
> exec bit when committed from Windows. `chmod +x install.sh` (or run
> `bash install.sh`).

> **⚠️ Pitfall — setcap "No such file or directory: target/release/dashboard".**
> The workspace `Cargo.toml` had `default-members = ["Portofino"]`, so a bare
> `cargo build --release` only built Portofino and never produced the dashboard
> binary. **Fixed** in the installer (uses `cargo build --release --workspace`).
> Manual unblock if you hit an old checkout:
> ```bash
> cargo build --release --workspace
> sudo setcap 'cap_net_bind_service=+ep' target/release/dashboard
> RUSTYTOOLZ_PORT=80 ./target/release/dashboard
> ```

---

## Troubleshooting Reference — every wall we hit

| Symptom | Cause | Fix |
|---|---|---|
| Ubuntu installer shows no free space / no "Add GPT Partition" | Raw *unallocated* space confuses Subiquity | Create the partition in **Windows Disk Management** first (Phase 2) |
| Ubuntu hangs "at cloud-init" on boot | Normal first-boot cloud-init | Wait / Enter; `sudo touch /etc/cloud/cloud-init.disabled` |
| FLARE install stalls — reboots into **Ubuntu** | GRUB defaults to Ubuntu; Boxstarter needs Windows | Set **GRUB default = Windows** (Phase 6) |
| FLARE package (Ghidra) fails mid-install | Wi-Fi not up yet after auto-login reboot | Re-run `install.ps1` until zero failures; prefer **wired Ethernet** |
| FLARE tools keep getting quarantined | Defender RTP re-enables on reboot (Home) | Tamper + RTP off, `Add-MpPreference -ExclusionPath 'C:\'` (RTP off *first*) |
| Clonezilla: `/home/partimag is not mounted normally, force?` | Ventoy NTFS **dirty flag** (unclean removal / Fast Startup) | **Never force a truly dirty vol.** `chkdsk D: /f` in Windows (authoritative), or `sudo ntfsfix -d /dev/sda1` in Linux. `powercfg /h off` stops recurrence |
| Clonezilla: **"Device or resource busy"** / **"Can't open blockdev"** / can't mount `/home/partimag` | Booted Clonezilla **from the Ventoy drive** → its ISO loopback locks the partition | **Boot Clonezilla from a SEPARATE USB stick.** (To RAM does *not* reliably free it under Ventoy) |
| Clonezilla image "no space left" / corrupted | Wrote to a **dirty** NTFS (drive was yanked live) | Clean the FS first (chkdsk / `ntfsfix -d`); never unplug the SSD live; clean-eject |
| `ntfsfix` didn't clear the dirty flag | Plain `ntfsfix` *sets* the check flag by design | Use **`ntfsfix -d`** (clears it), or Windows `chkdsk /f` |
| `install.sh` "Permission denied" | Exec bit lost on Windows-committed script | `chmod +x install.sh` or `bash install.sh` |
| setcap "No such file: target/release/dashboard" | `default-members` skipped the dashboard on bare build | `cargo build --release --workspace` (installer now fixed) |

---

## Key facts to remember

- **Image target device:** internal **`nvme0n1`** (1 TB). **Repository device:**
  Ventoy **`sda1`** (2 TB NTFS). Don't mix them up in Clonezilla.
- **One dedicated Clonezilla boot stick** — reuse for every save and restore.
- **Keep both images** (`baseline` + `golden`), never overwrite.
- **Clean-eject the Ventoy SSD every time**; keep it plugged through reboots.
- **Decrypt / detonate real samples only inside the analysis environment**, and
  image the box *before* the first sample touches it.
```
