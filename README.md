# bazzite-live-iso

Infraestructura para construir un **ISO live de Bazzite** en GitHub Actions con
[titanoboa](https://github.com/ublue-os/titanoboa).

> **Estado: en pausa.** Esperando un pendrive de 16 GB, que vuelve innecesario
> todo esto. Ver [Cómo retomar](#cómo-retomar).

## Por qué existe este repo

Objetivo original: probar Bazzite en una PC de escritorio **sin pendrive**,
arrancando un ISO desde el disco duro con GRUB (loopback).

Eso choca con un problema: los ISOs que Bazzite publica en `download.bazzite.gg`
son imágenes de **instalador Anaconda**, no live. Arrancan directo al
instalador, sin sesión de prueba. Y los ISOs live sí existen, pero no se
publican en ningún lado descargable.

## Hallazgos verificados

Todo esto está comprobado, no asumido:

| Hallazgo | Cómo se verificó |
|---|---|
| El ISO "stable" publicado es un instalador, no live | Montado y leído su `EFI/BOOT/grub.cfg`: solo entradas *Install*, con `inst.stage2=` |
| Ese ISO está basado en Fedora 41 (viejo) | Kernel `6.16.12-100.fc41` dentro del initrd |
| Rebajarlo no aporta nada | `Content-Length` del servidor idéntico al archivo local, byte a byte |
| **`iso-scan` está soportado** — GRUB loopback es viable | Initrd descomprimido (XZ, 326 MB): contiene `usr/sbin/iso-scan` y `var/lib/dracut/hooks/cmdline/31-parse-iso-scan.sh` |
| Los ISOs live de la CI de Bazzite expiran | Retención de 5 días; último build exitoso 2026-08-02, artifacts ya vencidos. De 1041 artifacts del repo, ninguno es un ISO vivo |
| Construirlo requiere la pipeline completa | `examples/bazzite/src/build.sh` de titanoboa: cambio de kernel a vanilla, `dracut-live`, livesys-scripts, Anaconda, flatpaks, `grub2-efi-x64-cdboot` |

## Lo que este workflow ya resuelve

Tres builds, tres problemas distintos, todos corregidos:

1. **`image not known`** — titanoboa monta la imagen como *image volume* de
   podman y no la descarga sola. Se agregó un `podman pull` explícito.
2. **`image not known` otra vez**, con la imagen ya bajada — titanoboa corre
   `sudo podman run`, o sea **rootful**, y podman mantiene almacenes separados
   para rootless y root. El pull tenía que ser con `sudo`.
3. **`Missing /usr/lib/bootc-image-builder/iso.yaml`** — pendiente. La imagen
   `ghcr.io/ublue-os/bazzite:stable` no trae ese archivo.

Con los dos primeros arreglados, titanoboa corre 10 minutos y genera el
squashfs correctamente (4,5 GB comprimidos desde 10,6 GB, 190.743 archivos)
antes de fallar en el punto 3.

También quedó medido: tras la limpieza el runner tiene **107-119 GB libres**,
espacio de sobra. Y no hacen falta runners privilegiados.

## Lo que falta

Derivar una imagen propia que incluya la configuración, siguiendo
[`examples/bazzite`](https://github.com/ublue-os/titanoboa/tree/main/examples/bazzite):

```dockerfile
FROM ghcr.io/ublue-os/bazzite:stable
COPY ./src /src
RUN --mount=type=cache,target=/var/tmp/libdnf5,id=libdnf5 /src/build.sh
RUN rm -fr /src
```

Construida con `sudo podman build --cap-add sys_admin --security-opt label=disable --squash`,
y después pasarle esa imagen local a titanoboa.

El `iso.yaml` mínimo:

```yaml
label: "Bazzite-Live"
grub2:
  timeout: 10
  entries:
    - name: "Bazzite Live ISO"
      linux: "/images/pxeboot/vmlinuz quiet rhgb root=live:CDLABEL=Bazzite-Live enforcing=0 rd.live.image"
      initrd: "/images/pxeboot/initrd.img"
```

Estimado: 2-4 ciclos de 30-70 minutos.

## Cómo retomar

### Con pendrive (recomendado)

Este repo deja de hacer falta. Se graba directamente el ISO instalador con
Rufus (ya descargado en `~/Downloads/rufus-4.15p.exe`) en modo **DD Image**,
partición **GPT** / destino **UEFI**.

Queda igual sin sesión de prueba — para eso hay que construir el ISO live o
usar el instalador y probar sobre el sistema ya instalado.

### Sin pendrive

1. Vendorizar `examples/bazzite/` de titanoboa en `src/`
2. Agregar el paso de `podman build` antes de titanoboa
3. Apuntar `image-ref` a la imagen derivada local
4. Bajar el artifact y **verificar `iso-scan` en su initrd antes de tocar el arranque**
5. Instalar GRUB2 en la ESP y agregar la entrada con
   `iso-scan/filename=/ruta/al.iso`

> ⚠️ La partición EFI de la máquina destino tiene **100 MB** y Windows ya usa
> parte. Es el punto más ajustado de todo el plan.

## Hardware de referencia

Ryzen 5 2400G · RX 580 (Polaris) · 16 GB RAM · UEFI · sin BitLocker ·
virtualización **desactivada** en BIOS · SSD 240 GB (GPT) + HDD 1 TB (MBR).

La RX 580 es Polaris: **SteamOS oficial no la soporta** (Valve validó RDNA2+),
por eso Bazzite y no SteamOS.

## Uso

Actions → *Build Bazzite Live ISO* → **Run workflow**.

| `image_ref` | Variante |
|---|---|
| `ghcr.io/ublue-os/bazzite:stable` | Escritorio KDE, AMD/Intel (por defecto) |
| `ghcr.io/ublue-os/bazzite-deck:stable` | Modo consola tipo Steam Deck |
| `ghcr.io/ublue-os/bazzite-gnome:stable` | Escritorio GNOME |
