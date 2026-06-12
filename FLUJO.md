# Flujo de NeonatoX Bootstrap

## Invocación

```
neonatox-bootstrap [--lfs <ruta>] [opciones] <comando>
```

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config dinámica, paquetes core |
| `kde` | chroot (`chroot $LFS`) | Escritorio KDE + common |
| `gnome` | chroot (`chroot $LFS`) | Escritorio GNOME + common |
| `xfce` | chroot (`chroot $LFS`) | Escritorio XFCE + common |

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Raíz destino |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo FS para fstab |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `-p, --root-password <pass>` | — | Contraseña root (vía chpasswd) |
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
      locale.conf           # locale es_US.UTF-8
      polkit-1/rules.d/
        00-mount-internal.rules
      profile.d/
        extrapaths.sh
        readline.sh
        umask.sh
        i18n.sh
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
2. **Archivos estáticos** — `cp -a bootstrap/data/etc/. → $LFS/etc/`
3. **Symlinks del sistema** — `ld-linux`, `awk`, `sh`, `mtab`
4. **Construir nhopkg** — `git clone` → `meson setup build` → `ninja install`
5. **Archivos dinámicos** — `fstab` (blkid), `hosts`/`hostname` (DMI+random), timezone (host), `machine-id`
6. **Paquetes core** — `nhopkg --root $LFS -U && -S core base-extra grub`

Todo se ejecuta directamente sobre `$LFS` usando `nhopkg --root`.

## Flujo: DE (kde / gnome / xfce)

Los tres comandos DE ejecutan dentro de chroot en `$LFS`:

1. **chroot_prepare** — monta VFS
2. **Desktop common** — `nhopkg -S desktop-common`, `systemctl enable lightdm`, `nhopkg --install-group ttf-fonts`, `nhopkg -S libpwquality pam`, `passwd root`
3. **Paquetes del DE** — según el comando:
   - KDE: `nhopkg -S kde + kde-extra`
   - GNOME: `nhopkg -S gnome + gnome-extra; nhopkg -r qt5 qt6`
   - XFCE: `nhopkg -S xfce + xfce-extra; nhopkg -r qt6`
4. **Cleanup** — `rm -rf /usr/src/* /var/nhopkg/cache/* /tmp/*`
5. **chroot_cleanup** — desmonta VFS

## Diagrama

```
┌──────────────────────────────────────┐
│         neonatox-bootstrap           │
│  Argumentos → LFS, hostname, pass... │
└──────────┬───────────────────────────┘
           │
    ┌──────┴──────────────┐
    ▼                     ▼
┌─────────┐     ┌──────────────────┐
│  core   │     │  kde/gnome/xfce  │
│ (host)  │     │    (chroot)      │
│         │     │                  │
│ 1. mkdir│     │ 1. chroot_prepare│
│ 2. cp   │     │ 2. desktop-common│
│ 3. symln│     │ 3. enable lightdm│
│ 4. nhop │     │ 4. ttf-fonts     │
│ 5. dyn  │     │ 5. pam+passwd    │
│ 6. pkgs │     │ 6. nhopkg -S DE  │
└─────────┘     │ 7. cleanup       │
                │ 8. chroot_cleanup│
                └──────────────────┘
```

## Dependencias entre comandos

- `core` debe ejecutarse **primero** (desde el host).
- `kde` / `gnome` / `xfce` se ejecutan **después** desde el host,
  asumen que `core` ya pobló `$LFS` con la jerarquía, nhopkg y la BD.

## Notas

- `nhopkg --root "$LFS"` se usa solo en `core` para paquetes sin
  postinstall triggers relevantes.
- `chroot_run "nhopkg -S"` se usa en DE commands para que los
  disparadores postinstall corran en el destino.
- `systemctl` requiere que el target tenga systemd instalado.
- `passwd root` usa `chpasswd` si se provee `--root-password`;
  si se omite, se salta con un aviso.
- El cleanup final elimina `/usr/src/*`, `/var/nhopkg/cache/*`,
  `/tmp/*` dentro del chroot.

## Problemas conocidos

- `drkonqi` requiere `gdb` (no declarado como dep)
- `transmission-qt` requiere `qt6-core` (no declarado)
- `vim` tiene dependencia errónea con `gpm`
- `gnome-control-center` necesita `udisks2` (corregir dep)
- Reinstalar `nhopkg -S pam` tras crear usuarios para permisos
  correctos de `libpwquality`
