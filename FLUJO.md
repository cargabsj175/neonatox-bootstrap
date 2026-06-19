# Flujo de NeonatoX Bootstrap

## Invocación

```
neonatox-bootstrap [--lfs <ruta>] [opciones] <comando>
```

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config dinámica, paquetes core |
| `kde` | host → chroot (`chroot_run`) | Escritorio KDE + desktop-common |
| `gnome` | host → chroot (`chroot_run`) | Escritorio GNOME + desktop-common |
| `xfce` | host → chroot (`chroot_run`) | Escritorio XFCE + desktop-common |
| `user` | host → chroot | Crear usuario (`useradd`, `groupadd`, `chpasswd`) |
| `grub` | host → chroot | Instalar GRUB (`grub-install` + `grub-mkconfig`) |
| `chroot` | host → chroot | Shell interactivo dentro de `$LFS` |

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Raíz destino |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo FS para fstab |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `--no-cleanup` | — | Salta `rm -rf /usr/src/* /tmp/*` |
| `--pack-dir <dir>` | `bootstrap/packs/` | Listas de paquetes |
| `-h, --help` | — | Muestra la ayuda |

## Estructura de archivos

```
neonatox-bootstrap          # ejecutable (bash)
bootstrap/
  data/
    etc/
      resolv.conf           # nameservers fijos
      passwd                # usuarios del sistema
      group                 # grupos del sistema
      inputrc               # config readline
      shells                # shells disponibles
      bashrc                # bashrc global
      profile               # profile global
      os-release            # ID del sistema
      lsb-release           # ID LSB
      polkit-1/rules.d/
        00-mount-internal.rules
      profile.d/
        extrapaths.sh
        readline.sh
        umask.sh
        i18n.sh
      default/
        grub                    # vars para grub-mkconfig
      skel/
        .bash_profile
        .bashrc
        .profile
  packs/
    core                    # 61 paquetes base
    base-extra              # 5 paquetes extra
    desktop-common          # 13 paquetes comunes DE
    kde                     # 78 paquetes KF6+Plasma
    kde-extra               # 13 paquetes apps KDE
    gnome                   # 13 paquetes GNOME core
    gnome-extra             # 11 paquetes apps GNOME
    xfce                    # 21 paquetes XFCE4
    xfce-extra              # 15 paquetes apps XFCE
01-base-neonatox.sh         # referencia original (intacto)
02-corespacks               # referencia original (intacto)
FLUJO.md                    # este archivo
AGENTS.md                   # guía para asistentes IA
testing_neoanatox_btrfs     # notas de workflow btrfs + systemd-nspawn
```

## Helpers de chroot

Tres funciones permiten ejecutar comandos dentro del entorno destino
sin depender de systemd-nspawn:

| Helper | Descripción |
|--------|-------------|
| `chroot_prepare` | Monta `/proc`, `/dev`, `/sys`, `/run` en `$LFS` |
| `chroot_run <cmd>` | Ejecuta `chroot "$LFS" /bin/bash -c "<cmd>"` |
| `chroot_cleanup` | Desmonta VFS en orden inverso |

## Flujo: core (host)

1. **Jerarquía de directorios** — `mkdir -pv` y symlinks (`/bin → usr/bin`, etc.)
2. **Archivos estáticos** — `cp -a bootstrap/data/etc/. → $LFS/etc/`, `skel/ → $LFS/root/`
3. **Symlinks del sistema** — `awk → gawk`, `sh → bash`, `mtab`
4. **Construir nhopkg** — `git clone` → `meson setup build` → `ninja install` → `cp -a DESTDIR/* $LFS/`
5. **Archivos dinámicos** — `fstab` (blkid), `hosts`/`hostname` (DMI+random), timezone (host), `machine-id`
6. **Paquetes core** — `nhopkg --root $LFS -U` (poblar BD), `-RS core base-extra`, `-RS grub --no-check-deps`, `-RS libpwquality pam`
7. **Triggers** — `chroot_prepare → run_triggers → chroot_cleanup`
8. **Sentinel** — se crea `$LFS/var/nhopkg/.core-installed`

Todo se ejecuta directamente sobre `$LFS` usando `nhopkg --root`. Los triggers
se ejecutan dentro de chroot para que los postinstall scripts corran en el
entorno destino.

## Flujo: DE (kde / gnome / xfce)

Los tres comandos DE se ejecutan desde el **host** y usan `chroot_run`
para operar dentro de `$LFS`. Cada comando asegura que `core` y
`desktop-common` estén instalados antes de proceder.

### Desktop common (shared)

1. `nhopkg --root $LFS -RS desktop-common`
2. **chroot_prepare** → `chroot_run "systemctl enable lightdm"` → **run_triggers** → **chroot_cleanup**
3. Se crea `$LFS/var/nhopkg/.desktop-common-installed`

La contraseña de root ya no se gestiona aquí. Ahora se usa:

```
neonatox-bootstrap user --username root -p <contraseña>
```

### Paquetes del DE

Se ejecutan desde el host (`nhopkg --root $LFS`):

| DE | Paquetes | Remociones |
|----|----------|------------|
| KDE | `-RS kde kde-extra` | — |
| GNOME | `-RS gnome gnome-extra` | `-Rr qt5 qt6` |
| XFCE | `-RS xfce xfce-extra` | `-Rr qt6` |

Luego, dentro de chroot: **chroot_prepare → run_triggers → cleanup → chroot_cleanup**.

## Diagrama

```
┌─────────────────────────────────────────┐
│           neonatox-bootstrap            │
│    Argumentos → LFS, hostname, ...      │
└──────────┬──────────────────────────────┐
           │                              │
     ┌─────┴──────────┐       ┌───────────┴──────────┐
     ▼                ▼       ▼                      ▼
┌──────────┐ ┌────────────┐ ┌──────────┐   ┌────────────────┐
│   core   │ │  user/grub │ │  DEs     │   │    chroot      │
│  (host)  │ │ host→chroot│ │ host→ch  │   │   interactivo  │
│          │ │            │ │   root   │   │                │
│ 1. mkdir │ │ Verificar  │ │ 1. deskt │   │ chroot_prepare │
│ 2. cp    │ │ sentinel   │ │ 2. common│   │ shell          │
│ 3. symln │ │ chroot_prep│ │ 3. prep  │   │ chroot_cleanup │
│ 4. nhopkg│ │ useradd /  │ │ 4. light │   └────────────────┘
│ 5. dyn   │ │ grub-inst  │ │ 5. dm en │
│ 6. pkgs  │ │ chpasswd / │ │ 6. trigs │
│ 7. trigs │ │ grub-mkcfg │ │ 7. clean │
│ 8. sentl │ │ triggers   │ │ 8. nhopk │
└──────────┘ │ cleanup     │ │ 9. -RS DE│
             └─────────────┘ │10. trigs │
                             │11. clean │
                             │12. clean │
                             └──────────┘
```

## Dependencias entre comandos

- `core` debe ejecutarse **primero** (desde el host).
- `kde` / `gnome` / `xfce` asumen que `core` ya pobló `$LFS`; cada
  uno verifica el sentinel y si falta, ejecuta `core` y `desktop-common`
  automáticamente.
- `user` y `grub` requieren `core` instalado (verifican el sentinel).
- `chroot` requiere `core` instalado (verifica el sentinel).

## Notas

- `nhopkg --root "$LFS"` se usa para operaciones que no requieren
  postinstall triggers (instalación de paquetes desde el host).
- `chroot_prepare` / `chroot_run` / `chroot_cleanup` se usan para
  ejecutar triggers postinstall y comandos systemd dentro del destino.
- `systemctl` requiere que el target tenga systemd instalado.
- `user --username root -p <pass>` reemplaza el antiguo `--root-password`.
- `grub` detecta el dispositivo automáticamente desde `/etc/fstab`;
  legacy por defecto, usar `--efi` para UEFI.
- Existen archivos **sentinel** (`/var/nhopkg/.core-installed`,
  `/var/nhopkg/.desktop-common-installed`) para evitar reinstalaciones.
- El cleanup final elimina `/usr/src/*`, `/var/nhopkg/cache/*`,
  `/tmp/*` dentro del chroot.
- Comando `chroot` — entra en un shell interactivo dentro de `$LFS`
  (requiere `core` instalado).

## Problemas conocidos

- `drkonqi` requiere `gdb` (no declarado como dep)
- `transmission-qt` requiere `qt6-core` (no declarado)
- `vim` tiene dependencia errónea con `gpm`
- `gnome-control-center` necesita `udisks2` (corregir dep)
- Reinstalar `nhopkg -S pam` tras crear usuarios para permisos
  correctos de `libpwquality`
