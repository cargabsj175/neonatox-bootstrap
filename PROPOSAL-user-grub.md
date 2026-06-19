# PROPOSAL: Creación de usuarios e instalación de GRUB

## Resumen

Tres cambios ejecutables desde el host, todos operan dentro de chroot
en `$LFS`:

1. **`user`** — Crear un usuario del sistema (nombre, fullname, grupos, shell, password),
   incluyendo `root` (reemplaza `--root-password` global).
2. **`grub`** — Instalar GRUB en el dispositivo detectado en fstab.
3. **Eliminar `--root-password`** de las opciones globales; su función
   la asume `user --username root -p <pass>`.

---

## 1. Comando `user`

### Propuesta

```
neonatox-bootstrap [--lfs <dir>] user [opciones]
```

Opciones del subcomando:

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-u, --username <name>` | **requerido** | Nombre de usuario (login) |
| `-f, --fullname <str>` | `""` | Nombre completo (GECOS) |
| `-g, --groups <lista>` | `users` | Grupos adicionales separados por coma (ej: `wheel,audio,video`) |
| `-s, --shell <ruta>` | `/bin/bash` | Shell de login |
| `-p, --password <pass>` | — | Contraseña (vía `chpasswd`). Si se omite, se salta. |
| `--no-create-home` | — | No crear `/home/<username>` |
| `--uid <num>` | auto | UID específico |
| `--gid <num>` | auto | GID del grupo primario |

### Comportamiento

1. Verifica que `core` esté instalado (sentinel).
2. Prepara chroot (`chroot_prepare`).
3. Si `--username root`:
   - Solo ejecuta `chpasswd` para root (no `useradd`, root ya existe).
   - Salta validación de grupos, shell, home, uid/gid.
4. Si `--username` es otro:
   - Valida grupos contra `$LFS/etc/group`:
     Los existentes se asignan. Los inexistentes se informan al usuario
     y se ignoran.
   - `groupadd` (si el grupo primario no existe).
   - `useradd` con los parámetros dados.
5. Si `--password` se provee: `chpasswd` para el usuario.
6. Ejecuta triggers.
7. Limpia chroot (`chroot_cleanup`).

### Consideraciones

- `useradd` y `groupadd` deben venir del paquete `shadow` (ya incluido en core).
- `chpasswd` también es de `shadow`.
- No se crea archivo sentinel; el comando es idempotente (si el usuario
  ya existe, `useradd` falla y se informa).
- `user --username root -p <pass>` reemplaza el antiguo `--root-password`.

### Futuro posible

- `--ssh-authorized-keys <ruta>` — copiar claves públicas al usuario.
- Múltiples usuarios desde archivo.

---

## 2. Comando `grub`

### Propuesta

```
neonatox-bootstrap [--lfs <dir>] grub [opciones]
```

Opciones:

| Opción | Default | Descripción |
|--------|---------|-------------|
| `-d, --device <dev>` | auto (mismo que fstab) | Dispositivo donde instalar GRUB |
| `--efi` | — | Activar modo UEFI (`x86_64-efi`) |
| `--efi-dir <dir>` | `/boot` | Directorio de la partición EFI |
| `--reinstall` | — | Pasar `--reinstall` a `grub-install` |

### Comportamiento

1. Verifica que `core` esté instalado.
2. Prepara chroot.
3. Detecta automáticamente el dispositivo desde `/etc/fstab` del target:
   - Lee la línea de `/` en `$LFS/etc/fstab`.
   - Si es `UUID=<uuid>`, resuelve a `/dev/disk/by-uuid/<uuid>` (vía `readlink -f`).
   - Si es un device directo (`/dev/sda3`), extrae la base (`/dev/sda`).
   - Si `--device` se provee, se usa ese valor directamente.
4. Ejecuta dentro del chroot:
   - **Legacy** (por defecto):
     ```
     grub-install <device>
     ```
   - **UEFI** (si `--efi`):
     ```
     grub-install --target=x86_64-efi --efi-directory=<efi-dir> --bootloader-id=NeonatoX
     ```
5. `grub-mkconfig -o /boot/grub/grub.cfg`
6. Limpia chroot.

### Consideraciones

- `grub` ya se instala durante `core` (`-RS grub --no-check-deps`).
- Para UEFI podría requerir montar la partición EFI. El usuario debe
  asegurarse de que esté montada en `$LFS/<efi-dir>` antes de ejecutar
  el comando.
- La detección del dispositivo base desde fstab usa `sed` para recortar
  el número de partición (ej: `/dev/sda3` → `/dev/sda`).

---

## 3. Eliminar `--root-password` global

### Cambios

- Quitar `-p, --root-password` del parsing de argumentos y de `show_help`.
- En `install_desktop_common()`, eliminar el bloque que ejecuta
  `chpasswd` para root:

```bash
# Eliminar esto:
if [[ -n "${ROOT_PASSWORD:-}" ]]; then
    echo "root:$ROOT_PASSWORD" | chroot "$LFS" chpasswd
else
    echo "  [saltado] passwd root — use --root-password o ejecútelo manualmente"
fi
```

### Razón

La contraseña de root ahora se define explícitamente con:

```
neonatox-bootstrap user --username root -p <contraseña>
```

Esto separa responsabilidades: `core`/`desktop-common` instalan el
sistema base, `user` gestiona usuarios y contraseñas.

---

## 4. Integración en `neonatox-bootstrap`

Ambos comandos se añaden al `case` principal:

```bash
case "$MODE" in
    core|kde|gnome|xfce|chroot|user|grub) ...
```

### Help

Actualizar `show_help` — quitar `--root-password` y añadir los nuevos
comandos:

```
Comandos:
  core            Instalar sistema base
  kde/gnome/xfce  Instalar escritorio
  user            Crear usuario del sistema
  grub            Instalar GRUB en dispositivo detectado
  chroot          Entrar en chroot interactivo
```

### FLUJO.md

- Añadir secciones para `user` y `grub` en la tabla de comandos.
- Eliminar `--root-password` de la tabla de opciones.
- Actualizar el diagrama de flujo y la nota sobre `passwd root`.

---

## 5. Consideraciones adicionales

### Parsing de argumentos por subcomando

Actualmente el script tiene un solo loop de parsing. Con `user` y `grub`
aparecen opciones propias (`-u`, `-g`, `-p`, `--efi`, etc.) que deben
consumirse después del subcomando. Propuesta:

1. El loop principal parsea solo opciones globales (`-L`, `--hostname`, etc.)
   y detecta el `MODE`.
2. Tras el `case "$MODE"`, cada función de subcomando recibe los args
   restantes y los parsea con `shift` y su propio while.

Esquema:

```bash
# main
case "$MODE" in
    user) shift; cmd_user "$@" ;;
    grub) shift; cmd_grub "$@" ;;
    ...
esac
```

```bash
cmd_user() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -u|--username) USERNAME="$2"; shift 2 ;;
            -p|--password) PASSWORD="$2"; shift 2 ;;
            ...)
        esac
    done
}
```

### `/etc/default/grub`

`grub-mkconfig` lee `/etc/default/grub`. Se crea como archivo estático
en `bootstrap/data/etc/default/grub`:

```bash
# NeonatoX GRUB config
GRUB_DEFAULT=0
GRUB_TIMEOUT=15
GRUB_TIMEOUT_STYLE=menu
GRUB_DISTRIBUTOR="Neonatox"
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX=""
GRUB_DISABLE_OS_PROBER=true
GRUB_GFXMODE=auto
GRUB_TERMINAL=console
```

## 6. Decisiones tomadas (ex-pendientes)

- **[ok]** Resolver dispositivo base desde UUID en fstab — se resuelve
  igual que en `install_core` (reusar lógica de detección), o vía
  `readlink -f /dev/disk/by-uuid/<uuid>`.
- **[ok]** Montar VFS — `chroot_prepare` ya lo hace, no requiere cambio.
- **[ok]** `user` crea el grupo primario homónimo por defecto (comportamiento
  de `useradd`). Solo se omite si se pasa `--gid`.
- **[no]** Flag `--sudo` — no se implementa por ahora.
