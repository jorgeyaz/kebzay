# CLAUDE.md — Proyecto KEBZAY

## Contexto del proyecto

Este es un proyecto web completo para **KEBZAY**, una empresa de maquinaria usada e importada.

**Jorge no tiene ningún conocimiento técnico.** El agente debe hacer TODO de forma autónoma: escribir el código, hacer el deploy, verificar que funciona, y resolver problemas sin requerir intervenciones manuales. Si hay algo que Jorge sí tiene que hacer (como pegar una clave en un sitio web), explicárselo en pasos numerados, simples y sin jerga técnica.

**Regla de oro:** Siempre verificá que los cambios funcionan antes de decirle a Jorge que está listo. Si algo puede hacerse automáticamente, hacelo.

---

## Protocolo de documentación — OBLIGATORIO

Cada vez que el agente haga un cambio importante, **debe actualizar este CLAUDE.md en la misma sesión**, antes de terminar. Esto incluye:

- Cualquier cambio en la infraestructura (SSH, tokens, URLs, claves)
- Librerías agregadas, cambiadas o eliminadas
- Bugs resueltos (qué era, por qué pasaba, cómo se resolvió)
- Bugs conocidos o pendientes
- Decisiones de arquitectura (por qué se eligió X sobre Y)
- Credenciales o accesos nuevos

**El objetivo es que este archivo sea siempre suficiente para que cualquier agente nuevo pueda retomar el trabajo desde cero, sin necesitar contexto de conversaciones anteriores.**

---

## Acceso a GitHub — SSH (configurado 2025-05-10)

El acceso a GitHub se hace por SSH. **No usar tokens HTTPS** (caducan y dan problemas).

- **Clave SSH:** `~/.ssh/kebzay_github` (privada) / `~/.ssh/kebzay_github.pub` (pública)
- **Clave pública registrada en GitHub** bajo el nombre `MacBook Jorge` en la cuenta `jorgeyaz`
- **Host configurado en** `~/.ssh/config`:
  ```
  Host github.com
    IdentityFile ~/.ssh/kebzay_github
    AddKeysToAgent yes
  ```
- **Verificar que funciona:** `ssh -T git@github.com` → debe responder `Hi jorgeyaz!`

### Comando para deployar cambios (usar siempre este)

```bash
git -C ~/Documents/kebzay add . && git -C ~/Documents/kebzay commit -m "descripcion" && git -C ~/Documents/kebzay push
```

**Nota:** La carpeta local del proyecto es `~/Documents/kebzay/` (no `~/Downloads/kebzay/`).

### Si SSH falla en una sesión nueva

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
ssh -T git@github.com
```

---

## Estructura del proyecto

### Repositorios GitHub

- **Cuenta GitHub:** `jorgeyaz`
- **Repo KEBZAY:** `git@github.com:jorgeyaz/kebzay.git` → publicado en `jorgeyaz.github.io/kebzay`
- **Repo Claudio Izsak:** `git@github.com:jorgeyaz/catalogo.git` → publicado en `jorgeyaz.github.io/catalogo`

### Archivos locales del proyecto KEBZAY

```
~/Documents/kebzay/
├── CLAUDE.md         # Este archivo — documentación del proyecto
├── index.html        # Menú principal (PIN: kebzay2026)
├── catalogo.html     # Catálogo de subasta
├── inventario.html   # App de inventario de maquinaria (~1900 líneas)
└── item.html         # Ficha pública de ítem (accesible por QR sin PIN)
```

---

## Backend — Supabase

- **URL:** `https://hxlechhhwkcxpexuqsgx.supabase.co`
- **Publishable key:** `sb_publishable_BntSAqmx7BfcPGDRL5XDmg_-aAk3xm4`
- **Organización:** kebzay
- **Nota:** El proyecto anterior (`gjsybqqwwwjntkobpodg`) fue eliminado por inactividad en el plan gratuito (2026-05-10). Este es el proyecto de reemplazo.

### Tablas

| Tabla | Descripción |
|-------|-------------|
| `kebzay_inventario` | Ítems del inventario de maquinaria |
| `kebzay_sesiones` | Sesiones guardadas del catálogo de subasta |

### Esquema de `kebzay_inventario`

```sql
id text PRIMARY KEY
codigo text                  -- Ej: GAS-AI-001
categoria text               -- Ej: gastronomia
subcategoria text            -- Ej: maquinaria
nombre text
descripcion text
estado text                  -- Bueno / Regular / Malo / Dado de baja
precio numeric               -- Precio de venta en $ ARS
cantidad integer             -- Stock actual
photos jsonb                 -- Array de base64 (fotos comprimidas)
baja_motivo text             -- Venta / Rotura / Desguace / Otro
baja_fecha date
baja_nota text
historial_bajas jsonb        -- Array de movimientos de baja
historial_ingresos jsonb     -- Array de movimientos de ingreso
created_at timestamptz
updated_at timestamptz
```

---

## App de inventario (`inventario.html`)

### Librerías usadas

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jsbarcode@3.11.6/dist/JsBarcode.all.min.js"></script>
```

**Importante — librería QR:** Usar siempre `qrcode-generator@1.4.4`. Se probó `qrcode@1.5.3` pero no tiene build para browser (la ruta CDN `/build/qrcode.min.js` no existe). No cambiar esta librería.

### Funcionalidades implementadas

- **Categorías y subcategorías** con código automático (ej: `GAS-AI-001`)
- **Fotos** (hasta 4 por ítem, comprimidas a 600px calidad 0.6)
- **Precio en $ ARS**, cantidad en stock
- **Estados:** Bueno / Regular / Malo / Dado de baja
- **Ingreso de stock** (↑ Ingreso) con motivo, fecha, nota e historial
- **Baja de stock** (↓ Baja) parcial o total; al llegar a 0 → "Dado de baja"
- **Historial de bajas** permanente
- **Panel de valorizado** por categoría/subcategoría con exportación PDF
- **Búsqueda** por texto y filtros
- **Selección de ítems** para generar PDF catálogo
- **Etiquetas para Zebra** con QR + código de barras, tamaño configurable
- **Ficha pública** por QR (`item.html?id=GAS-AI-001`) sin PIN
- **Guardado:** localStorage como primario, Supabase como sincronización

### Categorías del inventario

```
1. Gastronomía → Acero inox, Maquinaria, Muebles, Vajilla y útiles
2. Panificados → Maquinaria, Carros y útiles
3. Frío → Heladeras, Equipos de frío, Cámaras, Paneles de cámaras
4. Cárnicos → Maquinaria, Carros y útiles
5. Logística → Racks, Máquinas de movimiento
```

### QR en etiquetas — historial del bug (resuelto 2025-05-10)

**Problema:** Al generar el PDF de etiquetas, todos los QRs decodificaban al mismo ítem.

**Causa raíz:** Se estaba usando `pdf.rect()` de jsPDF para dibujar el QR como vectores directamente en el PDF. jsPDF comparte el estado de fill entre páginas y los rectángulos se sobreescribían internamente.

**Solución aplicada:** Cada QR se dibuja primero en un `<canvas>` HTML independiente, luego se convierte a PNG (`canvas.toDataURL('image/png')`) y se inserta en el PDF con `pdf.addImage(..., 'NONE')` usando un alias único por ítem (`qr-0`, `qr-1`, etc.). Esto garantiza que cada página del PDF tiene su propia imagen PNG independiente.

**Lo que NO funcionó (no volver a intentar):**
- `pdf.rect()` directo en jsPDF → sobreescribe estado entre páginas
- Google Charts API para generar QR → caché del servidor devuelve siempre el mismo
- `qrcode@1.5.3` vía CDN → no tiene build para browser
- `addImage` con alias sin canvas independiente → jsPDF reutiliza la imagen

---

## App catálogo de subasta (`catalogo.html`)

- Carga de lotes con fotos (hasta 4), descripción, unidades
- Descripción automática con IA via Cloudflare Worker (`claude-proxy.jorge-99c.workers.dev`)
- Generación de PDF catálogo
- Guardado en localStorage + Supabase (`catalogo_sesiones`)
- Backup/restore de sesiones por título de remate
- Export/Import JSON
- Fix cámara Android (input centralizado)
- Fix PDF: Android → descarga directa / iOS → Web Share API

---

## App catálogo Claudio Izsak (`jorgeyaz.github.io/catalogo`)

Misma funcionalidad que el catálogo KEBZAY pero con identidad de Claudio Izsak Remates. PIN: `ci2026`. Tabla Supabase: `catalogo_sesiones`. Repo separado: `git@github.com:jorgeyaz/catalogo.git`.

---

## Convenciones importantes

- **Precios siempre en $ ARS**, nunca USD
- **Fotos comprimidas** a 600px calidad 0.6 antes de guardar
- **PIN de acceso:** `kebzay2026` (compartido entre index, inventario y catálogo)
- **Autenticación:** via `sessionStorage` key `kebzay_pin_ok` — se valida solo en `index.html`; las demás páginas verifican esa key y redirigen al index si no está
- **localStorage key del inventario:** `kebzay_inventario_v2`
- **Upsert Supabase:** `Prefer: resolution=merge-duplicates,return=minimal`
- **Deploy:** verificar siempre con `curl -s https://jorgeyaz.github.io/kebzay/inventario.html | grep "texto_esperado"` antes de reportar éxito
- **Editar `inventario.html`:** tiene ~1900 líneas, usar siempre str_replace quirúrgico, nunca reescribir el archivo completo

---

## Instrucciones para el agente

1. **Hacé el trabajo vos.** Jorge no sabe de código. Si hay que editar un archivo, editalo. Si hay que deployar, deployá. Si hay que verificar, verificá con curl.
2. **Actualizá este CLAUDE.md** cada vez que resuelvas un bug, cambies infraestructura, o tomes una decisión importante. No esperes a que Jorge lo pida.
3. **Verificá siempre** que el cambio llegó a GitHub Pages antes de decir que está listo.
4. **Si algo no funciona** después de 2 intentos, cambiá el approach completamente.
5. **Cuando le expliques algo a Jorge**, usá lenguaje simple, pasos numerados, y evitá la jerga técnica.
6. **Antes de empezar cualquier tarea**, leé este archivo completo para entender el estado actual del proyecto.
