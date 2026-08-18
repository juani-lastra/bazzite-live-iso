# bazzite-live-iso

Construye un **ISO live de Bazzite** en GitHub Actions usando
[titanoboa](https://github.com/ublue-os/titanoboa).

## Para qué

Los ISOs que Bazzite publica en `download.bazzite.gg` son imágenes de
**instalador** (Anaconda): arrancan directo al instalador, sin sesión de
prueba. Este workflow genera en cambio un ISO **live**, que levanta un
escritorio completo para verificar hardware antes de instalar nada.

Además parte de la imagen actual de `ghcr.io`, bastante más nueva que el
último ISO publicado.

## Uso

Actions → *Build Bazzite Live ISO* → **Run workflow**.

El ISO queda como artifact (`bazzite-live-iso`) junto a su SHA256,
disponible 7 días.

### Variantes

Cambiando `image_ref` al lanzarlo:

| Imagen | Para qué |
|---|---|
| `ghcr.io/ublue-os/bazzite:stable` | Escritorio KDE, AMD/Intel (por defecto) |
| `ghcr.io/ublue-os/bazzite-deck:stable` | Modo consola tipo Steam Deck |
| `ghcr.io/ublue-os/bazzite-gnome:stable` | Escritorio GNOME |

## Nota sobre el espacio

El runner de GitHub trae ~25 GB libres, insuficiente para la imagen
descomprimida más el ISO de salida. Por eso el primer paso vacía el runner
antes de construir.
