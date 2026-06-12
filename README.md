# NeonatoX Bootstrap

![Bash](https://img.shields.io/badge/bash-4EAA25?logo=gnubash&logoColor=fff)
![Meson](https://img.shields.io/badge/meson-0288D1?logo=meson&logoColor=fff)
![Ninja](https://img.shields.io/badge/ninja-28B0E7?logo=ninja&logoColor=fff)
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
git clone https://github.com/cargabsj175/neonatox-nhopkg.git
```

```bash
sudo ./neonatox-bootstrap -L /mnt core
```

```bash
sudo systemd-nspawn -D /mnt
# dentro del contenedor:
# passwd root
# exit
```

```bash
sudo ./neonatox-bootstrap -L /mnt -p mipassword kde
```

### Argumentos

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-L, --lfs <dir>` | `$LFS` o `/mnt` | Directorio raíz de destino |
| `--hostname <nombre>` | auto (DMI) | Nombre del equipo |
| `--device <disp>` | auto-detect | Dispositivo para fstab |
| `--uuid <uuid>` | auto-detect | UUID para fstab |
| `--fstype <tipo>` | auto-detect | Tipo de sistema de archivos |
| `-z, --timezone <zona>` | host o `America/Caracas` | Zona horaria |
| `-l, --locale <locale>` | `es_US.UTF-8` | Locale del sistema |
| `-p, --root-password <pass>` | — | Contraseña root |
| `--no-cleanup` | — | Salta limpieza post-instalación |
| `--pack-dir <dir>` | `bootstrap/packs/` | Listas de paquetes |

### Comandos

| Comando | Contexto | Descripción |
|---------|----------|-------------|
| `core` | host (`nhopkg --root`) | Sistema base: directorios, nhopkg, config, paquetes core |
| `kde` | chroot (`chroot $LFS`) | Escritorio KDE + common |
| `gnome` | chroot (`chroot $LFS`) | Escritorio GNOME + common |
| `xfce` | chroot (`chroot $LFS`) | Escritorio XFCE + common |

## Estructura

```
neonatox-bootstrap          # ejecutable principal (bash)
bootstrap/
  data/etc/                  # archivos de configuracion estaticos
    {passwd,group,profile,bashrc,os-release,...}
    polkit-1/rules.d/
    profile.d/
    skel/
  packs/                     # listas de paquetes (uno por linea)
    {core,base-extra,desktop-common,kde,gnome,xfce,...}
01-base-neonatox.sh         # referencia original (intacto)
02-corespacks               # referencia original (intacto)
```

## Requisitos

- Linux con `chroot`, `mount --bind`, `git`, `meson`, `ninja`
- `sudo` para `blkid` y `mount`
- Conexión a internet para clonar nhopkg y descargar paquetes

## Recursos

- [NeonatoX](https://neonatox.vegnux.com)
- [nhopkg](https://github.com/cargabsj175/neonatox-nhopkg)
