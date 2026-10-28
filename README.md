# Antojo Express

Tienda de comidas rápida completa (catálogo, carrito, checkout, pedidos y panel administrativo) construida con Next.js 15, React 19, Prisma y SQLite. Pensada para operar en Colombia con precios en COP.

> **Aviso de demostración:** el catálogo, los precios, las promociones, las zonas de domicilio y los tiempos de entrega incluidos por el seed son **datos de ejemplo**. No representan un negocio real. Reemplázalos desde el panel antes de publicar. El sitio no inventa número de WhatsApp, redes sociales, reseñas ni datos de contacto: todos esos campos están vacíos por defecto y se configuran en el panel.

## Características

### Tienda pública
- Inicio con hero, productos populares, promociones activas, combos, reseñas aprobadas (si existen), FAQ y CTA.
- Menú con búsqueda, filtro por categoría y ordenamiento sin recargar la página.
- Modal de producto con opciones obligatorias/opcionales, precio dinámico, notas, favoritos y control de agotados.
- Carrito persistente en `localStorage`, restaurante validado de nuevo contra la base de datos.
- Checkout con entrega o recogida, zonas y barrios, mínimo de compra, horarios reales del negocio, métodos de pago habilitados y notas.
- Registro real de pedidos en base de datos con número único `AE-YYMMDD-XXXXXX` y enlace público de seguimiento.
- Botón flotante de WhatsApp configurable y resumen prellenado del pedido mediante `wa.me`. **Abrir WhatsApp no confirma el pedido**: la confirmación se realiza desde el panel.
- Páginas de nosotros, contacto, FAQ, privacidad, términos, favoritos, 404 personalizada y volver arriba.
- Diseño responsive, foco visible, diálogos nativos accesibles y respeto por `prefers-reduced-motion`.

### Panel administrativo (`/admin`)
- Autenticación propia con contraseña bcrypt, sesión JWT en cookie `httpOnly` y control de acceso en middleware y servidor.
- Dashboard con indicadores calculados desde datos reales.
- Pedidos: detalle, cambio de estado con historial y marcas de tiempo.
- CRUD de productos, categorías, combos, promociones y reseñas.
- Zonas de domicilio, métodos de pago, horarios, datos del negocio, redes y WhatsApp.
- Bandeja de mensajes de contacto.
- Informes con rango de fechas y exportación CSV protegida.
- Cambio de contraseña del administrador.

## Stack

| Área | Tecnología |
| --- | --- |
| Framework | Next.js 15 (App Router), React 19, TypeScript estricto |
| Estilos | Tailwind CSS 3.4 |
| Base de datos | Prisma 6 + SQLite |
| Validación | Zod |
| Carrito y favoritos | Zustand con persistencia |
| Autenticación | bcryptjs + jose (JWT HS256) |
| Imágenes | next/image + sharp para las cargas del panel |

## Requisitos

- Node.js 20 o superior (probado con Node 22/24).
- npm.

## Puesta en marcha

```bash
npm install
cp .env.example .env
```

Edita `.env` y completa los valores. `.env.example` deja `AUTH_SECRET` vacío a propósito: el login del panel no funcionará hasta que lo configures. Para generar un secreto de sesión válido:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Crea la base de datos, carga los datos de demostración y arranca:

```bash
npm run db:push
npm run db:seed
npm run dev
```

- Tienda: http://localhost:3000
- Panel: http://localhost:3000/admin/login

### Credenciales de demostración

El seed crea el administrador con `ADMIN_EMAIL` y `ADMIN_PASSWORD`. Si no se definen, usa estos valores de ejemplo:

- Correo: `admin@antojoexpress.demo`
- Contraseña: `Antojo2026*`

Son **solo para la demo local**. Cámbialas antes de desplegar (el panel permite cambiar la contraseña en Configuración). No uses estas credenciales en producción.

## Variables de entorno

| Variable | Obligatoria | Descripción |
| --- | --- | --- |
| `DATABASE_URL` | Sí | Ruta de SQLite, por ejemplo `file:./dev.db`. |
| `AUTH_SECRET` | Sí | Secreto para firmar sesiones. Mínimo 32 caracteres; si falta o es corto, el login falla. |
| `NEXT_PUBLIC_SITE_URL` | Recomendada | URL pública sin barra final. Se usa en metadatos, canonical, sitemap y robots. En producción debe ser HTTPS. |
| `ADMIN_EMAIL` | Solo seed | Correo del administrador inicial. |
| `ADMIN_PASSWORD` | Solo seed | Contraseña del administrador inicial. |

`NEXT_PUBLIC_SITE_URL` se incorpora en tiempo de compilación: vuelve a construir después de cambiarla. `AUTH_SECRET` y las credenciales del seed nunca se envían al navegador.

## Base de datos

La base local está en `prisma/dev.db` (ignorada por git). El esquema incluye: administradores, categorías, productos y sus grupos de opciones, combos, promociones y productos participantes, zonas de domicilio, métodos de pago, configuración del negocio, clientes, pedidos, ítems, opciones, historial de estados, mensajes de contacto y reseñas.

Comandos:

```bash
npm run db:push    # sincroniza el esquema
npm run db:seed    # carga datos de demostración (puede ejecutarse varias veces)
npm run db:reset   # BORRA la base y vuelve a cargar el seed
```

`db:reset` es destructivo: elimina pedidos, clientes y mensajes. Úsalo solo en desarrollo.

El seed es idempotente para los datos base: no duplica registros existentes y no sobrescribe cambios hechos desde el panel. Los nombres de zonas incluyen `(demo)` y el aviso de demostración está activo por defecto; puedes desactivarlo en Configuración.

## Configuración del negocio

En **Admin → Configuración**:

- **Datos y contacto:** nombre, eslogan, descripción, historia, teléfono, correo y dirección. Déjalos vacíos si no son reales.
- **WhatsApp:** solo dígitos con código de país, por ejemplo `57XXXXXXXXXX` (7 a 15 dígitos). Si queda vacío, los botones de WhatsApp y el botón flotante no se muestran.
- **Redes sociales:** URLs completas. Solo se enlazan las que configures; también alimentan `sameAs` del JSON-LD.
- **Horarios:** se interpretan en `America/Bogota`. Si el local está cerrado, el checkout se bloquea y el servidor rechaza el pedido.
- **Zonas:** cada zona define barrios, costo de domicilio, pedido mínimo y tiempo estimado. Solo se aceptan barrios de zonas activas.
- **Métodos de pago:** solo los activos aparecen en el checkout. Son métodos **offline** (efectivo o transferencia): no hay pasarela ni simulación de pago aprobado.

### Flujo del pedido

1. El carrito se guarda en el navegador y se revalida contra la base de datos.
2. El checkout envía el pedido a `/api/orders`; el servidor recalcula precios, descuentos, opciones, stock, zona, mínimo, horario y método de pago dentro de una transacción.
3. Se registra el pedido y su historial. El cliente recibe un enlace público de seguimiento.
4. Si hay WhatsApp configurado, se muestra un enlace con el resumen. Abrirlo no cambia el estado del pedido.
5. El negocio confirma o cancela desde **Admin → Pedidos**; cada cambio queda en el historial con fecha y nota.

Las promociones se aplican solo si están activas en fecha, sobre productos participantes, con un único descuento (el mejor) por producto y sin acumulación. Los totales siempre se recalculan en el servidor.

## Scripts

| Script | Uso |
| --- | --- |
| `npm run dev` | Servidor de desarrollo. |
| `npm run build` | Genera Prisma y compila para producción. |
| `npm run start` | Sirve la compilación de producción. |
| `npm run db:push` | Sincroniza el esquema con la base. |
| `npm run db:seed` | Carga los datos de demostración. |
| `npm run db:reset` | Reinicia la base (destructivo). |

Para verificar tipos: `npx tsc --noEmit`.

## Seguridad

- El panel exige autenticación real; no se basa en ocultar rutas. Las sesiones se validan contra la base y comprueban que el usuario siga activo.
- Cookie de sesión `httpOnly`, `sameSite=lax` y `secure` en producción.
- Contraseñas con bcrypt (costo 12) y límite de intentos de login.
- Middleware protege `/admin/*` y `/api/admin/*`; las acciones sensibles vuelven a validar el administrador.
- Precios, cantidades, descuentos, totales, cobertura y métodos de pago se validan en el servidor.
- Los pedidos públicos usan un token aleatorio de 128 bits; no se expone el id interno.
- La exportación CSV y la carga de imágenes requieren sesión de administrador.
- No hay secretos ni credenciales embebidos en el código del cliente.

## Despliegue

La aplicación está preparada para un servidor Node con disco persistente:

```bash
npm ci
npm run db:push
npm run db:seed      # solo la primera vez o para una demo
npm run build
npm run start
```

Antes de publicar:

1. Configura `AUTH_SECRET` con un valor aleatorio largo y `NEXT_PUBLIC_SITE_URL` con el dominio HTTPS definitivo.
2. Cambia la contraseña del administrador y no conserves las credenciales de demo.
3. Reemplaza productos, precios, zonas, horarios y aviso de demo por datos reales.
4. Configura WhatsApp y redes solo si son reales.
5. Sirve el sitio detrás de HTTPS (por ejemplo, con un proxy inverso).

Consideraciones:

- **SQLite necesita disco persistente.** No es adecuada para despliegues serverless efímeros. Para producción con varias instancias, migra el datasource de Prisma a PostgreSQL/MySQL.
- **Imágenes subidas:** el panel guarda archivos en `public/uploads`. Ese directorio también debe ser persistente y respaldarse.
- **Copias de seguridad:** respalda la base (`prisma/dev.db`) y `public/uploads` de forma periódica.
- Para un proyecto real, sustituye `db push` por migraciones versionadas de Prisma.
- No hay envío automático de correos ni integración con pasarelas de pago; esas funciones no se simulan.

## Estructura

```
prisma/
  schema.prisma        Modelo de datos
  seed.ts              Datos de demostración claramente identificados
src/
  app/(site)/          Tienda pública
  app/admin/           Login y panel administrativo
  app/api/             Pedidos, contacto, informes CSV y carga de imágenes
  components/          UI de tienda, carrito y panel
  lib/                 Prisma, autenticación, validación y reglas de negocio
public/images/         Imágenes optimizadas de la demo
public/uploads/        Imágenes cargadas desde el panel (ignoradas por git)
scripts/               Optimización de imágenes
```

## SEO y rendimiento

- Metadatos, canonical, Open Graph y Twitter Card por página.
- `sitemap.xml` y `robots.txt` generados desde `NEXT_PUBLIC_SITE_URL`; las rutas privadas están excluidas.
- JSON-LD de restaurante construido únicamente con datos configurados. No se inventan reseñas, valoraciones, precios, direcciones ni teléfonos.
- Imágenes locales optimizadas con `next/image` y prioridad en el hero.
- Páginas privadas de pedido y checkout marcadas como `noindex`.
