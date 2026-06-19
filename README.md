# NeonatoX Bootstrap

![Bash](https://img.shields.io/badge/bash-4EAA25?logo=gnubash&logoColor=fff)
![systemd](https://img.shields.io/badge/systemd-052A46?logo=systemd&logoColor=fff)
![btrfs](https://img.shields.io/badge/btrfs-2D8659?logo=btrfs&logoColor=fff)
![Linux](https://img.shields.io/badge/linux-FCC624?logo=linux&logoColor=000)

Bootstrap scripts para [NeonatoX](https://neonatox.vegnux.com), una
distribución Linux desde cero. Crea la jerarquía del sistema de archivos,
compila e instala el gestor de paquetes `nhopkg`, genera la configuración
base, e instala los paquetes para los entornos de escritorio
KDE / GNOME / XFCE.

## Uso rápido

```bash
sudo ./neonatox-bootstrap -L /mnt core               # sistema base
sudo ./neonatox-bootstrap -L /mnt -p mipassword kde  # KDE (host → chroot)
```

### Argumentos

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino |
| `--hostname <nombre>` | auto (DMI+random) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo de sistema de archivos |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `-p, --root-password <pass>` | — | Contraseña root (vía chpasswd) |
| `--no-cleanup` | — | Salta `rm -rf /usr/src/* /var/nhopkg/cache/* /tmp/*` |
| `--pack-dir <dir>` | `bootstrap/packs/` | Listas de paquetes |

### Comandos

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config dinámica, paquetes core |
| `kde` | host → chroot (`chroot_run`) | Escritorio KDE + desktop-common |
| `gnome` | host → chroot (`chroot_run`) | Escritorio GNOME + desktop-common |
| `xfce` | host → chroot (`chroot_run`) | Escritorio XFCE + desktop-common |
| `chroot` | host → chroot | Shell interactivo dentro de `$LFS` |

## Estructura

```
neonatox-bootstrap          # ejecutable principal (bash)
bootstrap/
  data/etc/                 # archivos de configuración estáticos
    {passwd,group,profile,bashrc,os-release,inputrc,shells,...}
    polkit-1/rules.d/
    profile.d/
    skel/
  packs/                    # listas de paquetes (uno por línea)
    core                    # paquetes base del sistema
    base-extra              # paquetes extra del sistema
    desktop-common          # paquetes comunes a todos los DE
    kde / kde-extra         # KF6+Plasma y apps KDE
    gnome / gnome-extra     # GNOME core y apps
    xfce / xfce-extra       # XFCE4 y apps
01-base-neonatox.sh         # referencia original (intacto)
02-corespacks               # referencia original (intacto)
FLUJO.md                    # diagrama de flujo y detalle paso a paso
AGENTS.md                   # guía para asistentes IA
testing_neoanatox_btrfs     # notas de workflow btrfs + systemd-nspawn
```

## Requisitos

- Linux con `chroot`, `mount --bind`, `git`, `meson`, `ninja`
- `sudo` para `blkid` y `mount`
- Conexión a internet para clonar nhopkg y descargar paquetes

## Flujo

Ver [`FLUJO.md`](FLUJO.md) para el detalle paso a paso de cada comando,
helpers de chroot (`chroot_prepare`, `chroot_run`, `chroot_cleanup`),
diagrama de flujo y notas de ejecución.

## Workflow btrfs

Ver [`testing_neoanatox_btrfs`](testing_neoanatox_btrfs) para el flujo
con subvolúmenes btrfs, snapshots y systemd-nspawn.

## Recursos

- [NeonatoX](https://neonatox.vegnux.com)
- [nhopkg](https://github.com/cargabsj175/neonatox-nhopkg)
