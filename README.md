# APIC Perú — Web Platform

**Stack:** Next.js 16 (App Router) · React 19 · Tailwind CSS v4 · Turbopack · TypeScript

---

## Table of Contents

**Part I — Developer Technical Guide**
1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Directory Architecture](#3-directory-architecture)
4. [Quickstart & Commands](#4-quickstart--commands)
5. [Routing Map](#5-routing-map)
6. [Cart System Architecture](#6-cart-system-architecture)
7. [Image Asset Convention](#7-image-asset-convention)

**Parte II — Guía Comercial para el Equipo APIC**
8. [Manual de Autogestión del Catálogo (SSOT)](#8-manual-de-autogestión-del-catálogo-ssot)
9. [Guía de Temas Estacionales](#9-guía-de-temas-estacionales)
10. [Buenas Prácticas Finales](#10-buenas-prácticas-finales)

---

# Part I — Developer Technical Guide

## 1. Project Overview

APIC Perú is the commercial web platform for **APIC**, a Peruvian manufacturer
of audiovisual and event equipment (360° platforms, geodesic domes, photo
booths, LED structures, and related hardware). The application is built on
**Next.js 16** using the App Router, with **React 19** powering all client
interactivity and **Tailwind CSS v4** as the styling system, running on the
**Turbopack** compiler for fast local iteration and production builds.

The architecture is deliberately split into two concerns that never leak
into one another:

- **Code** (this repository's `.tsx`/`.ts` files) owns layout, interaction,
  state management, and rendering logic. It contains zero hardcoded catalog
  data.
- **Content** (a single file, `app/config.ts`) owns every editable fact about
  the business: products, pricing, clearance flags, contact details, and
  even the site's active color identity.

This separation means the APIC team can run the entire catalog, pricing, and
seasonal branding lifecycle without opening a single `.tsx` file.

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router, Client Components) |
| UI Library | React 19 |
| Styling | Tailwind CSS v4 (utility classes + dynamic CSS custom properties) |
| Dev/Build Compiler | Turbopack |
| Language | TypeScript |
| Cart State | Custom hook (`useCart`) backed by `localStorage` |
| Dynamic Routing | File-based `[id]` segments for product and combo pages |

## 3. Directory Architecture

```
app/
├── config.ts                      # SSOT — all editable site content lives here
├── layout.tsx                     # Root layout shared by every route
├── globals.css                    # Global styles (includes scroll-behavior: smooth)
├── page.tsx                       # Home: Hero, "Nosotros" section, Liquidación de Stock
│
├── hooks/
│   └── useCart.ts                 # Cart hook — localStorage-backed selection state
│
├── lib/
│   └── catalogCardHelpers.ts      # Shared card helpers: badges, pricing, spec differentiation
│
├── productos/
│   └── [id]/
│       └── page.tsx               # Product detail: gallery, dynamic spec sheet, warranties,
│                                   # WhatsApp CTA
│
└── combos/
    └── [id]/
        └── page.tsx               # Family dashboard: sidebar + combo grid
                                    # (accepts /combos/all as the unfiltered default)
```

**Key invariant:** `app/page.tsx`, `app/productos/[id]/page.tsx`, and
`app/combos/[id]/page.tsx` read 100% of their product/pricing/theme data
from `WEBSITE_CONFIG` and `THEME_CONFIG`, exported by `app/config.ts`. None
of them contain inline catalog data.

## 4. Quickstart & Commands

```bash
# 1. Install dependencies
npm install

# 2. Start the development server (Turbopack)
npm run dev
# → available at http://localhost:3000
# Edits to .tsx files and config.ts hot-reload instantly

# 3. Build for production
npm run build

# 4. Serve the production build locally
npm start
# → available at http://localhost:3000, serving the optimized build
```

## 5. Routing Map

| Route | Description |
|---|---|
| `/` | Home — corporate hero, "Nosotros" section, Liquidación de Stock grid |
| `/productos/[id]` | Single product detail page (`id` is numeric, unique across the catalog) |
| `/combos/[id]` | Family dashboard (`id` is the family's `id_seccion` slug, e.g. `plataforma-360`) |
| `/combos/all` | Same dashboard with no family preselected — defaults to the first family |

## 6. Cart System Architecture

`app/hooks/useCart.ts` is a standalone hook providing:

- `selectedItems` — the current cart contents (product names), held in state
  and persisted to `localStorage` so selections survive page reloads.
- `toggleItem(nombre)` — add/remove a product by name, callable from any
  catalog card across all three pages.
- The quotation summary modal reads `selectedItems` directly to build a
  pre-formatted WhatsApp message, sent to the number configured in
  `WEBSITE_CONFIG.numero_whatsapp`.

No component maintains its own parallel cart state — `useCart` is the single
source of truth for cart contents app-wide.

## 7. Image Asset Convention

Product photography is served from `/public/equipos/`, organized by family
and combo tier:

```
public/equipos/PLATAFORMA-360º/COMBO 1/PLATAFORMA 360_IMG 1 PLAT 01.jpg
public/equipos/PLATAFORMA-360º/COMBO 2/PLATAFORMA 360_IMG 1 PLAT 02.jpg
```

Each product references its images via `imagenes_galeria`, an array of paths
relative to `/public`. The first entry in the array is used as the card
thumbnail; any number of additional images populate the product gallery.

---

# Parte II — Guía Comercial para el Equipo APIC

*A partir de aquí, esta guía está escrita en español, dirigida al equipo de
APIC que administrará el catálogo y la identidad visual del sitio sin
necesidad de conocimientos de programación.*

## 8. Manual de Autogestión del Catálogo (SSOT)

Todo el catálogo vive dentro de un solo archivo: **`app/config.ts`**. Ahí
encontrarán `WEBSITE_CONFIG.secciones_catalogo`: una lista de **familias de
producto** (por ejemplo, "Plataforma 360" o "Domos"). Cada familia contiene
una lista interna de **productos** — uno por cada nivel Combo (Combo 1,
Combo 2, etc.). Cada Combo es una tarjeta independiente en la tienda, con su
propio precio, fotos y ficha técnica.

El archivo incluye, como referencia, el primer producto del catálogo
(Plataforma 360 — Combo 1) completamente comentado campo por campo. El resto
de los productos sigue exactamente ese mismo patrón.

### 8.1. Cómo actualizar un precio en soles

Cada producto tiene un arreglo llamado `especificaciones`: una lista de
textos con el formato `'Clave: Valor'`. El precio es la línea que empieza
con `"Costo:"` o `"Precio:"`:

```
especificaciones: [
  'Peso: 25 kg',
  'Medida: 1 metro de diámetro',
  // ...
  'Costo: 2200 soles', // ← editar directamente aquí
],
```

**Esta es la única línea que debe cambiarse.** La ficha técnica que el
cliente ve en la pestaña "Ficha Técnica" del producto se genera
automáticamente a partir de esta misma lista — no existe un segundo lugar
donde el precio deba repetirse ni mantenerse sincronizado a mano.

> **Límite de seguridad de TypeScript:** el precio es siempre un texto
> (string) entre comillas simples `'...'`. No se debe borrar ninguna de las
> dos comillas, ni la coma al final de la línea — ambas son necesarias para
> que el archivo siga siendo válido y el sitio no deje de compilar.

### 8.2. Cómo agregar una nueva familia de equipos

Para dar de alta una familia nueva dentro de `secciones_catalogo`, se copia
la siguiente estructura completa, se pega al final de la lista de familias
existentes, y se reemplazan los valores:

```
{
  id_seccion: 'nombre-slug-sin-espacios',   // usado en la URL /combos/nombre-slug-sin-espacios
  titulo_seccion: 'Nombre Visible de la Familia',
  productos: [
    {
      id: 30,   // debe ser único en TODO el catálogo — ningún otro producto puede repetir este número
      nombre: 'Nombre Visible de la Familia — Combo 1',
      sku: 'ABC-EVT-001-C1',
      categoria: 'Nombre Visible de la Familia',   // debe coincidir con 'titulo_seccion' de arriba
      especificaciones: [
        'Clave 1: Valor 1',
        'Clave 2: Valor 2',
        'Costo: 0000 soles',
      ],
      descripcion: 'Resumen corto del producto.',
      descripcion_larga: 'Texto comercial y persuasivo para la página de detalle.',
      imagenes_galeria: ['/equipos/CARPETA-DE-LA-FAMILIA/COMBO 1/foto-1.jpg'],
      sobre_la_marca: 'APIC es fabricante líder en soluciones audiovisuales...',
      garantias: [
        { pregunta: 'Garantía de fábrica APIC (1 año)', respuesta: 'SÍ' },
      ],
      en_oferta: false,
      precio_antes: null,
    },
    // más Combos de la misma familia, si corresponde
  ],
},
```

Una vez guardado el archivo, la nueva familia aparece **automáticamente** en
el Mega Menu, en el dashboard de combos (`/combos/[id]`), y en cualquier
cálculo del catálogo general. No es necesario editar `app/page.tsx` ni
ningún otro archivo del proyecto.

> **Zona técnica — no tocar:** al final de `config.ts`, después del
> comentario "ZONA TÉCNICA — NO EDITAR A PARTIR DE AQUÍ", el archivo combina
> automáticamente todas las familias en una sola lista interna
> (`CATALOGO_GENERAL`) que usan las páginas de producto y de combos para
> buscar cualquier artículo. Esa sección es lógica de programación, no datos
> de catálogo — modificarla puede romper la compilación del sitio.

### 8.3. Cómo gestionar una oferta de Liquidación de Stock, sin romper nada

Cada producto tiene dos campos dedicados exclusivamente a esto:

```
en_oferta: true,
precio_antes: 'Precio Antes: 2600 soles',
```

- **Para poner un producto en oferta:** cambiar `en_oferta` a `true`, y
  escribir el precio anterior en `precio_antes`, respetando el mismo formato
  de texto que ya usan los demás productos (`'Precio Antes: XXXX soles'`,
  entre comillas simples).
- **Para quitar un producto de oferta:** cambiar `en_oferta` a `false`, y
  `precio_antes` a `null` (sin comillas — `null` es una palabra especial que
  significa "sin valor", no un texto).

El sitio calcula el porcentaje de descuento automáticamente, comparando
`precio_antes` contra el precio vigente del producto, y lo muestra como una
insignia (por ejemplo, `-15%`) sobre la tarjeta, tanto en la sección
"Liquidación de Stock" del Home como en el detalle del producto.

**Para mantener la compilación del sitio sin errores**, respetar siempre:
- `en_oferta` solo acepta `true` o `false` (sin comillas, en minúsculas).
- `precio_antes` solo acepta un texto entre comillas simples, o la palabra
  `null` — nunca dejarlo vacío ni borrar la línea completa.

## 9. Guía de Temas Estacionales

Toda la identidad de color del sitio —botones, enlaces, insignias de oferta,
el brillo ambiental de las tarjetas "Nosotros" y los bordes del pie de
página— se controla desde **una sola variable**, ubicada en la parte
superior de `app/config.ts`:

```
export const TEMA_ACTIVO: keyof typeof THEME_SCHEMES = 'PRO_DEFAULT'
```

Para cambiar la paleta de colores de **todo el sitio, de forma instantánea y
en armonía total entre todas las páginas**, basta con reemplazar el texto
entre comillas por una de estas tres opciones (deben escribirse exactamente
así, con mayúsculas y guion bajo):

| Esquema | Identidad | Color primario |
|---|---|---|
| `'PRO_DEFAULT'` | Turquesa Corporativo — identidad de siempre | `#039dbf` |
| `'WARM_SEASON'` | Cobre Cálido Cinemático — campañas de verano / aniversario | `#c2703d` |
| `'HOLIDAY_PREMIUM'` | Carmín Profundo de Lujo — campañas navideñas / fin de año | `#9b1b30` |

No es necesario modificar ningún otro archivo del proyecto. Las tres páginas
principales (`app/page.tsx`, `app/productos/[id]/page.tsx`,
`app/combos/[id]/page.tsx`) leen sus colores a través de variables CSS
inyectadas dinámicamente a partir de `TEMA_ACTIVO` en tiempo de ejecución, de
modo que el cambio se propaga en cascada e instantáneamente a toda la
plataforma: botones, enlaces, insignias y bordes decorativos cambian juntos,
como un conjunto ya balanceado — nunca de forma aislada ni descoordinada.

Cada esquema define, además del color principal, un tono de hover, un
degradado radial para el brillo ambiental de las tarjetas "Nosotros", y un
tono dedicado para las insignias de oferta. Los tres esquemas fueron
diseñados como paletas completas para mantener contraste y legibilidad sin
importar cuál esté activo.

## 10. Buenas Prácticas Finales

- **No se requieren cambios de código** para actualizar precios, fotos,
  ofertas o el tema de color activo del sitio — todo vive en
  `app/config.ts`.
- Antes de publicar cambios grandes en producción, se recomienda revisar
  visualmente el Home, el detalle de un producto, y el dashboard de combos
  tras cualquier edición extensa en `config.ts`, para confirmar que los
  datos se muestran correctamente.
- Al copiar un producto o una familia como plantilla para crear uno nuevo,
  verificar siempre que el `id` del producto sea único en todo el catálogo —
  es el único campo donde un valor repetido puede causar comportamiento
  inesperado en el sitio.
- Si se agrega un cuarto esquema de color en el futuro, se recomienda seguir
  exactamente la misma estructura de campos que los tres esquemas
  existentes, para que el resto del sitio lo reconozca sin cambios
  adicionales de código.