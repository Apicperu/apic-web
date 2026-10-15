# APIC Perú — Plataforma Web

**Stack:** Next.js 16 (App Router) · React 19 · Tailwind CSS v4 · Turbopack · TypeScript

---

## Tabla de Contenidos

**Parte I — Guía Técnica para Desarrolladores**
1. [Resumen del Proyecto](#1-resumen-del-proyecto)
2. [Stack Tecnológico](#2-stack-tecnológico)
3. [Arquitectura de Directorios](#3-arquitectura-de-directorios)
4. [Instalación y Comandos](#4-instalación-y-comandos)
5. [Mapa de Rutas](#5-mapa-de-rutas)
6. [Arquitectura del Carrito](#6-arquitectura-del-carrito)
7. [Convención de Imágenes](#7-convención-de-imágenes)
8. [Despliegue y Analíticas](#8-despliegue-y-analíticas)

**Parte II — Guía Comercial para el Equipo APIC**
9. [Manual de Autogestión del Catálogo (SSOT)](#9-manual-de-autogestión-del-catálogo-ssot)
10. [Guía de Temas Estacionales](#10-guía-de-temas-estacionales)
11. [Buenas Prácticas Finales](#11-buenas-prácticas-finales)

---

# Parte I — Guía Técnica para Desarrolladores

## 1. Resumen del Proyecto

APIC Perú es la plataforma web comercial de **APIC**, fabricante peruano de
soluciones audiovisuales y equipos para eventos (plataformas 360°, domos
geodésicos, cabinas fotográficas, estructuras LED y hardware relacionado). La
aplicación está construida sobre **Next.js 16** con el App Router, usa
**React 19** para toda la interactividad del cliente y **Tailwind CSS v4**
como sistema de estilos. En desarrollo y en compilación se apoya en el
compilador **Turbopack**.

La arquitectura separa dos responsabilidades que nunca se mezclan:

- **Código** (los archivos `.tsx` y `.ts` del repositorio): define el diseño,
  la interacción, el manejo de estado y la lógica de renderizado. No contiene
  ningún dato de catálogo escrito a mano.
- **Contenido** (un único archivo, `app/config.ts`): contiene todo lo que el
  negocio puede editar. Es decir, productos, precios, marcas de oferta, datos
  de contacto y la identidad de color activa del sitio.

Gracias a esta separación, el equipo de APIC puede gestionar el catálogo, los
precios y la imagen estacional de la marca sin abrir ningún archivo `.tsx`.

## 2. Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Framework | Next.js 16 (App Router, Client Components) |
| Librería de UI | React 19 |
| Estilos | Tailwind CSS v4 (clases utilitarias + variables CSS dinámicas) |
| Compilador / Servidor de desarrollo | Turbopack |
| Lenguaje | TypeScript |
| Estado del carrito | Contexto de React (`CartProvider` + hook `useCart`) con persistencia en `localStorage` |
| Rutas dinámicas | Segmentos `[id]` basados en archivos para producto y combo |
| Tipografías | Geist Sans y Geist Mono (`next/font/google`) |
| Hosting y analíticas | Vercel + `@vercel/analytics` |

## 3. Arquitectura de Directorios

```
app/
├── config.ts                      # SSOT — todo el contenido editable del sitio
├── layout.tsx                     # Layout raíz: tipografías, metadata, CartProvider y Analytics
├── globals.css                    # Estilos globales (incluye scroll-behavior: smooth)
├── page.tsx                       # Home: Hero, sección "Nosotros", Liquidación de Stock
│
├── hooks/
│   └── useCart.ts                 # CartProvider + hook useCart (selección persistida en localStorage)
│
├── lib/
│   └── catalogCardHelpers.ts      # Helpers compartidos: insignias, precios, specs diferenciadoras
│
├── productos/
│   └── [id]/
│       └── page.tsx               # Detalle de producto: galería, ficha técnica dinámica,
│                                   # garantías y CTA de WhatsApp
│
└── combos/
    └── [id]/
        └── page.tsx               # Dashboard por familia: sidebar + grilla de combos
                                    # (acepta /combos/all como vista por defecto)
```

**Invariante clave:** `app/page.tsx`, `app/productos/[id]/page.tsx` y
`app/combos/[id]/page.tsx` leen el 100% de sus datos de producto, precio y
tema desde `WEBSITE_CONFIG` y `THEME_CONFIG`, exportados por `app/config.ts`.
Ninguna de ellas contiene datos de catálogo escritos directamente en su código.

**Layout raíz (`app/layout.tsx`):** define las tipografías Geist, la metadata
del sitio (título, descripción e íconos), envuelve toda la aplicación en
`<CartProvider>` y renderiza el componente `<Analytics />` de Vercel
únicamente cuando `NODE_ENV === 'production'`.

## 4. Instalación y Comandos

```bash
# 1. Instalar las dependencias
npm install

# 2. Iniciar el servidor de desarrollo (Turbopack)
npm run dev
# → disponible en http://localhost:3000
# Los cambios en archivos .tsx y en config.ts se recargan al instante

# 3. Compilar para producción
npm run build

# 4. Servir la compilación de producción de forma local
npm start
# → disponible en http://localhost:3000, sirviendo la versión optimizada
```

## 5. Mapa de Rutas

| Ruta | Descripción |
|---|---|
| `/` | Home: hero corporativo, sección "Nosotros" y grilla de Liquidación de Stock |
| `/productos/[id]` | Detalle de un producto (`id` numérico, único en todo el catálogo) |
| `/combos/[id]` | Dashboard de una familia (`id` es el `id_seccion` de la familia, p. ej. `plataforma-360`) |
| `/combos/all` | El mismo dashboard sin familia preseleccionada: activa la primera por defecto |

## 6. Arquitectura del Carrito

El carrito vive en `app/hooks/useCart.ts` y se compone de dos piezas:

- **`CartProvider`**: contexto de React que envuelve toda la aplicación desde
  `app/layout.tsx`. Mantiene la selección en estado, la carga desde
  `localStorage` una sola vez después del montaje en el cliente y la guarda
  cada vez que cambia. También se sincroniza entre pestañas del navegador
  mediante los eventos `storage` y `focus`.
- **`useCart()`**: hook que expone `selectedItems` (la lista de productos
  seleccionados, por nombre) y `toggleItem(nombre)` para añadir o quitar un
  producto desde cualquier tarjeta de las tres páginas. Si se usa fuera de
  un `CartProvider`, lanza un error explícito.

El modal "Resumen de Cotización" lee `selectedItems` y arma un mensaje
preformateado que se envía por WhatsApp al número configurado en
`WEBSITE_CONFIG.numero_whatsapp`.

Ningún componente mantiene su propio estado de carrito paralelo:
`CartProvider` es la única fuente de verdad del contenido del carrito en toda
la aplicación.

## 7. Convención de Imágenes

La fotografía de producto se sirve desde `/public/equipos/`, organizada por
familia y nivel de combo:

```
public/equipos/PLATAFORMA-360º/COMBO 1/PLATAFORMA 360_IMG 1 PLAT 01.jpg
public/equipos/PLATAFORMA-360º/COMBO 2/PLATAFORMA 360_IMG 1 PLAT 02.jpg
```

Cada producto referencia sus imágenes en `imagenes_galeria`, un arreglo de
rutas relativas a `/public`. La primera imagen del arreglo se usa como
miniatura en las tarjetas; el resto completa la galería del producto.

## 8. Despliegue y Analíticas

El sitio se publica en **Vercel**. Cada `git push` a la rama conectada genera
un nuevo despliegue automático.

Las analíticas de visitas usan `@vercel/analytics`:

- El paquete debe figurar en las `dependencies` del `package.json`. Si falta,
  se instala con `npm i @vercel/analytics`.
- `<Analytics />` se renderiza solo en producción (`NODE_ENV === 'production'`),
  por lo que con `npm run dev` no se registran visitas.
- Para ver los datos, se debe habilitar Analytics en el panel del proyecto en
  Vercel (pestaña **Analytics**) y navegar por el sitio ya publicado. Los
  datos aparecen en aproximadamente 30 segundos.
- Los bloqueadores de anuncios pueden impedir el registro de visitas durante
  las pruebas.

---

# Parte II — Guía Comercial para el Equipo APIC

*Esta parte está dirigida al equipo de APIC que administrará el catálogo y la
identidad visual del sitio sin necesidad de conocimientos de programación.*

## 9. Manual de Autogestión del Catálogo (SSOT)

Todo el catálogo vive dentro de un solo archivo: **`app/config.ts`**. Ahí se
encuentra `WEBSITE_CONFIG.secciones_catalogo`: una lista de **familias de
producto** (por ejemplo, "Plataforma 360" o "Domos"). Cada familia contiene
una lista interna de **productos**, uno por cada nivel Combo (Combo 1,
Combo 2, etc.). Cada Combo es una tarjeta independiente en la tienda, con su
propio precio, fotos y ficha técnica.

El archivo incluye, como referencia, el primer producto del catálogo
(Plataforma 360 — Combo 1) comentado campo por campo. El resto de los
productos sigue exactamente ese mismo patrón.

### 9.1. Cómo actualizar un precio en soles

Cada producto tiene un arreglo llamado `especificaciones`: una lista de textos
con el formato `'Clave: Valor'`. El precio es la línea que empieza con
`"Costo:"` o `"Precio:"`:

```
especificaciones: [
  'Peso: 25 kg',
  'Medida: 1 metro de diámetro',
  // ...
  'Costo: 2200 soles', // ← editar directamente aquí
],
```

**Esta es la única línea que debe cambiarse.** La ficha técnica que el cliente
ve en la pestaña "Ficha Técnica" del producto se genera automáticamente a
partir de esta misma lista. No existe un segundo lugar donde el precio deba
repetirse ni mantenerse sincronizado a mano.

> **Límite de seguridad de TypeScript:** el precio es siempre un texto
> (string) entre comillas simples `'...'`. No se debe borrar ninguna de las
> dos comillas ni la coma al final de la línea. Ambas son necesarias para que
> el archivo siga siendo válido y el sitio no deje de compilar.

### 9.2. Cómo agregar una nueva familia de equipos

Para dar de alta una familia nueva dentro de `secciones_catalogo`, se copia la
siguiente estructura completa, se pega al final de la lista de familias
existentes y se reemplazan los valores:

```
{
  id_seccion: 'nombre-slug-sin-espacios',   // se usa en la URL /combos/nombre-slug-sin-espacios
  titulo_seccion: 'Nombre Visible de la Familia',
  productos: [
    {
      id: 30,   // debe ser único en TODO el catálogo: ningún otro producto puede repetirlo
      nombre: 'Nombre Visible de la Familia — Combo 1',
      sku: 'ABC-EVT-001-C1',
      categoria: 'Nombre Visible de la Familia',   // debe coincidir con 'titulo_seccion'
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
el Mega Menu, en el dashboard de combos (`/combos/[id]`) y en cualquier
cálculo del catálogo general. No es necesario editar `app/page.tsx` ni ningún
otro archivo del proyecto.

> **Zona técnica, no tocar:** al final de `config.ts`, después del comentario
> "ZONA TÉCNICA — NO EDITAR A PARTIR DE AQUÍ", el archivo combina
> automáticamente todas las familias en una sola lista interna
> (`CATALOGO_GENERAL`), que usan las páginas de producto y de combos para
> buscar cualquier artículo. Esa sección es lógica de programación, no datos
> de catálogo. Modificarla puede romper la compilación del sitio.

### 9.3. Cómo gestionar una oferta de Liquidación de Stock sin romper nada

Cada producto tiene dos campos dedicados exclusivamente a esto:

```
en_oferta: true,
precio_antes: 'Precio Antes: 2600 soles',
```

- **Para poner un producto en oferta:** cambiar `en_oferta` a `true` y escribir
  el precio anterior en `precio_antes`, respetando el mismo formato de texto
  que usan los demás productos (`'Precio Antes: XXXX soles'`, entre comillas
  simples).
- **Para quitar un producto de oferta:** cambiar `en_oferta` a `false` y
  `precio_antes` a `null` (sin comillas: `null` es una palabra especial que
  significa "sin valor", no un texto).

El sitio calcula el porcentaje de descuento automáticamente, comparando
`precio_antes` contra el precio vigente del producto, y lo muestra como una
insignia (por ejemplo, `-15%`) sobre la tarjeta en la sección "Liquidación de
Stock" del Home y en el detalle del producto.

**Para mantener la compilación sin errores, respetar siempre:**
- `en_oferta` solo acepta `true` o `false` (sin comillas, en minúsculas).
- `precio_antes` solo acepta un texto entre comillas simples o la palabra
  `null`. Nunca debe quedar vacío ni borrarse la línea completa.

## 10. Guía de Temas Estacionales

Toda la identidad de color del sitio (botones, enlaces, insignias de oferta,
el brillo ambiental de las tarjetas "Nosotros" y los bordes del pie de
página) se controla desde **una sola variable**, ubicada en la parte superior
de `app/config.ts`:

```
export const TEMA_ACTIVO: keyof typeof THEME_SCHEMES = 'PRO_DEFAULT'
```

Para cambiar la paleta de **todo el sitio de forma instantánea y con total
armonía entre páginas**, basta con reemplazar el texto entre comillas por una
de estas tres opciones (deben escribirse exactamente así, con mayúsculas y
guion bajo):

| Esquema | Identidad | Color primario |
|---|---|---|
| `'PRO_DEFAULT'` | Turquesa Corporativo: la identidad de siempre | `#039dbf` |
| `'WARM_SEASON'` | Cobre Cálido Cinemático: campañas de verano y aniversario | `#c2703d` |
| `'HOLIDAY_PREMIUM'` | Carmín Profundo de Lujo: campañas navideñas y de fin de año | `#9b1b30` |

No es necesario modificar ningún otro archivo del proyecto. Las tres páginas
principales (`app/page.tsx`, `app/productos/[id]/page.tsx` y
`app/combos/[id]/page.tsx`) leen sus colores mediante variables CSS que se
inyectan a partir de `TEMA_ACTIVO`. Por eso el cambio se propaga en cascada a
toda la plataforma: botones, enlaces, insignias y bordes decorativos cambian
juntos, como un conjunto balanceado y nunca de forma aislada.

Cada esquema define, además del color principal, un tono de hover, un
degradado radial para el brillo ambiental de las tarjetas "Nosotros" y un tono
dedicado para las insignias de oferta. Los tres esquemas fueron diseñados como
paletas completas para mantener el contraste y la legibilidad sin importar
cuál esté activo.

## 11. Buenas Prácticas Finales

- **No se requieren cambios de código** para actualizar precios, fotos,
  ofertas o el tema de color activo: todo vive en `app/config.ts`.
- Antes de publicar cambios grandes en producción, se recomienda revisar
  visualmente el Home, el detalle de un producto y el dashboard de combos tras
  cualquier edición extensa de `config.ts`, para confirmar que los datos se
  muestran correctamente.
- Al copiar un producto o una familia como plantilla para crear uno nuevo,
  verificar siempre que el `id` del producto sea único en todo el catálogo. Es
  el único campo donde un valor repetido puede causar comportamiento
  inesperado en el sitio.
- Si en el futuro se agrega un cuarto esquema de color, se recomienda seguir
  exactamente la misma estructura de campos que usan los tres existentes,
  para que el resto del sitio lo reconozca sin cambios adicionales de código.