# PIA — Esencia Hogar (app de gestión)

Web app para dos usuarios (el dueño y su socia) de un emprendimiento que vende piezas 3D, cerámica y aromas. Gestiona stock (filamentos, insumos, productos terminados), productos con receta y costo, combos, pedidos y precios por canal de venta. Idioma de la app y del usuario: español (Argentina).

## Arquitectura
- Un solo archivo: `index.html` (HTML + CSS + JS vanilla). `prototipo.html` es un prototipo viejo con datos de ejemplo, no se usa.
- Publicado en GitHub Pages desde `main` (repo público `pereyrapablojavier/esenciahogarpia`). Cada push a `main` se publica solo.
- Datos en Supabase, proyecto `omsqeklrbstoirljrhjb` (región us-west-2). Cliente `supabase-js` v2 por CDN. La clave publicable está en el código a propósito (es pública); la seguridad la da RLS.
- Login con email + contraseña. Todas las tablas tienen RLS con `is_allowed()`, que solo deja pasar los emails de la tabla `allowed_users`. El rol `authenticated` necesita GRANT explícito en cada tabla o función nueva.

## Modelo de datos (schema public)
- `config` (fila id=1: costo_hora, margen_objetivo, packaging), `canales` (comisión % y cargo fijo), `filamentos`, `insumos`.
- `productos`: nombre + `variante` (las variantes son filas distintas con el mismo nombre), `linea` (3D, Cerámica, Aromas, Combo), `stock`, `activo` (archivar = `activo=false`, no se borra), `tags text[]` compartidos por toda la familia de variantes, `es_combo`.
- `producto_filamentos`, `producto_insumos` (receta), `precios` (producto × canal), `combo_componentes` (combo → producto, cantidad).
- `pedidos` (guardan comisión, fijo y packaging del momento) y `pedido_items` (guardan precio_unit, costo_unit y `de_stock`). Así cambiar precios o costos después no altera el historial.

## Lógica en la base (funciones Postgres, security invoker)
- `crear_pedido`, `editar_pedido`, `cancelar_pedido`: descuentan/devuelven stock de forma atómica. Si falta stock, el pedido queda Pendiente con `faltan` ("a producir"); si alcanza, queda Preparando. Cancelar devuelve stock solo si el estado era Pendiente o Preparando.
- `tomar_stock`, `devolver_stock`, `costo_producto`: un combo no tiene stock propio; su disponibilidad es el mínimo entre sus componentes (stock / cantidad), y vender un combo descuenta de cada componente. No hay combos anidados.
- `guardar_producto` (9 argumentos) y `guardar_combo`. Queda una versión vieja de `guardar_producto` con 8 argumentos sin uso (se puede borrar con confirmación del dueño).
- Cambios de esquema: usar migraciones y funciones con `create or replace` (evitar DROP, pide confirmación). Probar lógica de pedidos en un bloque DO que termina en `raise exception` para que haga rollback.

## Cliente (index.html)
- Estado global `S`; `act(fn)` ejecuta, recarga con `load()` y vuelve a pintar con `render()`; `ok(promise)` lanza error si Supabase devuelve error.
- Funciones útiles: `nom(p)`, `activos()`, `costo(p)` (incluye combos), `stockEff(p)`, `neto`, `gananciaPedido` (resta el packaging del pedido), `openModal/closeModal`, `toast`.
- Los buscadores redibujan solo la lista (`drawProds`, `drawArch`, `drawStock`, `drawPed`, `drawPrecios`) para no perder el foco al escribir.
- Estética de la marca: colores #6e6043 / #9b9a9a / crema, tipografías Niramit y Rubik.

## Probar y publicar
- Revisar sintaxis del script: extraer el contenido de `<script>` y correr `node --check`.
- Pruebas rápidas con un DOM simulado en Node (stubs de `supabase`, `document`, `window`) que cargan el script y llaman `render()`.
- Publicar: `git add`, `git commit`, `git push origin main`. Después de un deploy, recargar con Ctrl+F5.

## Pendiente / ideas
- Las comisiones de los canales (Mercado Libre 14% + $1500, Tienda Nube 3,5%, Instagram 0) son valores de ejemplo: el dueño debe poner las reales en Precios.
- Posible: respaldo/exportación a Excel, packaging distinto por producto.
