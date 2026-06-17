# AGENTS.md — NeonatoX Bootstrap

## Commits

Solo hacer commit cuando el usuario lo indique explícitamente.

# AGENTS.md — NeonatoX Bootstrap

## What this repo is

Bootstrap scripts for a [NeonatoX](https://neonatox.vegnux.com) Linux
system: directory layout, `nhopkg` package manager, core config files,
and desktop environment packs (KDE/GNOME/XFCE). Run on a live
environment.

## Entry point

`neonatox-bootstrap` — single executable (bash). Static config files are
in `bootstrap/data/` (mirroring target `/`); package lists are in
`bootstrap/packs/` (one package per line). Original scripts
`01-base-neonatox.sh` and `02-corespacks` are left intact for reference.

## Usage

```
neonatox-bootstrap [--lfs <dir>] [options] core          # base system
neonatox-bootstrap [--lfs <dir>] [options] kde           # inside chroot
neonatox-bootstrap [--lfs <dir>] [options] gnome         # inside chroot
neonatox-bootstrap [--lfs <dir>] [options] xfce          # inside chroot
```

- `--lfs <dir>` / `-L <dir>` — target root (default: `$LFS` env or `/mnt`)
- `--hostname <name>` — skip DMI-based auto-generation
- `--device`, `--uuid`, `--fstype` — override fstab auto-detection
- `--timezone <zone>` / `-z` — set timezone (default: auto-detect from host)
- `--locale <locale>` / `-l` — set locale (default: `es_US.UTF-8`)
- `--root-password <pass>` / `-p` — set root password (skip if omitted)
- `--no-cleanup` — skip `rm -rf /usr/src/* /tmp/*` at end
- `--pack-dir <dir>` — custom pack list directory

Run `core` from the **host**. Run `kde`/`gnome`/`xfce` from the **host**
as well — they use `chroot_run` to execute commands inside `$LFS`,
so postinstall triggers run in the target environment.

## Architecture

- **`core`** creates filesystem hierarchy, copies `bootstrap/data/etc/*`,
  builds `nhopkg` (Meson+Ninja via
  <https://github.com/cargabsj175/neonatox-nhopkg>), generates dynamic
  config (`fstab`, `hostname`, `machine-id`, timezone), populates nhopkg
  DB, and installs core + base-extra package packs.
- **Desktop commands** install `desktop-common`, a DE pack, run
  `systemctl enable lightdm`, `passwd root`, fix PAM, and clean up.
  - GNOME additionally removes qt5/qt6; XFCE removes qt6.

## nhopkg (custom package manager)

- `nhopkg --root $LFS -U` — populate DB on target
- `nhopkg --root $LFS -S <pkg>` / `-r <pkg>`
- `nhopkg -S <pkg> --no-check-deps` — bypass dep checks
- `nhopkg --install-group ttf-fonts` — install a group

## btrfs workflow (testing_neoanatox_btrfs)

- Format: `mkfs.btrfs -L neonatox -f /dev/sda3`
- Create `@core` subvolume, mount with `-o subvol=@core`
- Snapshot → `@kde`, `@gnome`, `@xfce`
- Mount at `/var/lib/machines/<name>` for `systemd-nspawn`

## Known dependency issues (from 02-corespacks notes)

- `drkonqi` missing `gdb`
- `transmission-qt` missing `qt6-core`
- `vim` has erroneous `gpm` dependency
- `gnome-control-center` needs `udisks2` fix
- PAM reinstall (`nhopkg -S pam`) needed after user setup for
  `libpwquality` permissions
