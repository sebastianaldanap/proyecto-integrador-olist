# Revisión de la calidad de los datos (EDA): dataset público Brazilian E-Commerce

**Elaborado por:** Sebastian Aldana (fase EDA y calidad de datos) · **Fecha:** 19/11/2026

Este documento resume lo que se encontró al revisar los 9 archivos del dataset de Olist y **qué se hará con cada problema** al limpiar los datos en la capa Plata. Las reglas de este documento son las que se aplican en el proyecto.

## 1. Contexto

| Dato | Detalle |
|---|---|
| Dataset | Brazilian E-Commerce Public Dataset by Olist (Kaggle) |
| Archivos | 9 archivos CSV |
| Plataforma | Databricks Free Edition |
| Dónde están los datos | Catálogo `olist_proyecto`, schema `bronce` (una tabla por archivo) |
| Notebook del análisis | `notebooks/03_eda_calidad_datos` |
| Lenguaje | Python |

**Cómo se cargaron los datos:** en la capa Bronce todo se guardó como texto, sin cambiar nada, para tener una copia fiel del original. Los números y las fechas se convierten a su tipo correcto en la capa Plata.

## 2. Resumen

- Los 9 archivos se leyeron completos, sin errores de lectura y sin perder filas.
- Los identificadores de 7 tablas no se repiten, y 6 de las 7 conexiones entre tablas están completas (ninguna fila queda sin pareja).
- Se revisaron las principales columnas numéricas y se aplicó un método estadístico (rango intercuartílico) para detectar valores extremos.
- Los problemas son pocos en cantidad, salvo en dos tablas: `geolocation` (26% de las filas están repetidas) y `order_reviews` (547 pedidos tienen más de una reseña).
- **Ninguna regla borra pedidos ni ventas.** Cuando un dato es dudoso, se conserva y se marca con una columna de aviso.
- Se comprobó cuántos clientes reales hay (96,096), cuántos pedidos no tienen productos (775, ninguno entregado) y cuántos clientes y vendedores no tienen ubicación (279 y 7).
- Se comprobó que el total pagado coincide con el total de los productos en el 99.75% de los pedidos.

## 3. Cantidad de filas por tabla

| Tabla | Filas |
|---|---|
| `customers` (clientes) | 99,441 |
| `orders` (pedidos) | 99,441 |
| `order_items` (productos de cada pedido) | 112,650 |
| `order_payments` (pagos) | 103,886 |
| `order_reviews` (reseñas) | 99,224 |
| `products` (productos) | 32,951 |
| `sellers` (vendedores) | 3,095 |
| `geolocation` (ubicaciones) | 1,000,163 |
| `category_translation` (traducción de categorías) | 71 |

## 4. Qué se encontró y qué se hará

### 4.1 Lectura de los archivos

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 1 | Todas | Ninguna fila tuvo problemas de lectura. Databricks agregó a cada tabla una columna extra llamada `_rescued_data`, que está vacía en todas las filas | Quitar la columna `_rescued_data` en la capa Plata. Convertir a número las columnas numéricas revisadas (precios, fletes, pagos, cuotas, pesos, medidas y cantidad de fotos), porque en Bronce están como texto |

### 4.2 Datos faltantes

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 2 | `orders` | Faltan fechas de aprobación (160), de envío (1,783) y de entrega (2,965). Se explica casi por completo por el estado del pedido (los cancelados o aún en camino no tienen esas fechas), con 8 excepciones (ver punto 3) | Dejar esas fechas vacías |
| 3 | `orders` | 8 pedidos figuran como entregados (`delivered`) pero no tienen fecha de entrega | Conservarlos y agregar la columna de aviso `entregado_sin_fecha` |
| 4 | `order_reviews` | Muchas reseñas no tienen título (87,658) ni comentario (58,256), de 99,224. Escribirlos es opcional para el cliente | Dejarlos vacíos |
| 5 | `products` | 610 productos no tienen categoría, nombre, descripción ni fotos. Los 610 sí se vendieron (1,603 ventas, el 1.4% del total). Otros 2 productos no tienen peso ni medidas | Conservarlos y asignar la categoría `sin_categoria`. Dejar vacíos el peso y las medidas de los 2 productos |
| 6 | `products` | Dos columnas tienen un error de ortografía: `product_name_lenght` y `product_description_lenght` | Corregir el nombre a `length` |

### 4.3 Filas repetidas e identificadores

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 7 | 7 tablas | Los identificadores no se repiten: cliente, pedido, producto, vendedor, categoría, producto dentro de un pedido y pago dentro de un pedido | No requiere acción |
| 8 | `order_reviews` | El código de reseña (`review_id`) se repite en pedidos distintos: 789 códigos aparecen más de una vez. No hay filas idénticas. Por eso no sirve como identificador | No usar `review_id` para identificar reseñas. Usar el pedido |
| 9 | `order_reviews` | 547 pedidos tienen 2 o 3 reseñas (543 con 2 y 4 con 3). Si no se corrige, al unir pedidos con reseñas cada pedido saldría repetido y las ventas se contarían de más | **Dejar una reseña por pedido: la más reciente** (por fecha de creación; si empatan, la que se respondió después) |
| 10 | `geolocation` | 261,831 filas son copias exactas de otra (de 1,000,163 quedan 738,332 distintas) | Eliminar las copias |
| 11 | `geolocation` | Hay 19,015 códigos postales, con unas 39 ubicaciones cada uno. Si se une por código postal, cada cliente se multiplicaría | Dejar una sola ubicación por código postal (el promedio de sus puntos) |

### 4.4 Valores fuera de lo normal

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 12 | `geolocation` | 27 puntos están fuera de Brasil (latitudes que no corresponden o puntos en Argentina). Fernando de Noronha es una isla de Brasil y se considera válida | Excluir esos 27 puntos antes de calcular el promedio por código postal |
| 13 | `geolocation` | 4 códigos postales no tienen ningún punto válido (`78131`, `95130`, `18243`, `83252`). Los usa 1 cliente y ningún vendedor | Conservar al cliente y dejar su ubicación vacía, sin borrarlo |
| 14 | `order_items` | Los precios son válidos (no hay ceros ni negativos). El precio típico es de 74.99, el 1% más caro supera 889.99 y el máximo es 6,735. Los 10 más altos son razonables. El más llamativo es uno de `utilidades_domesticas` (6,735), pero su pedido fue entregado y se pagó completo (6,929.31 con el flete), por lo que se considera un valor real | Conservar todos y agregar la columna de aviso `precio_atipico` para precios sobre 889.99 |
| 15 | `order_items` | 383 ventas tienen flete en 0 (0.34%). Probablemente sean envíos gratis, aunque no se pudo confirmar | Conservarlas sin cambios |

### 4.5 Conexiones entre tablas

| # | Conexión | Qué se encontró | Qué se hará |
|---|---|---|---|
| 16 | Pedidos con clientes, productos de pedidos con pedidos, productos y vendedores, pagos con pedidos, reseñas con pedidos | Todas las conexiones están completas: ninguna fila apunta a algo que no existe | No requiere acción |
| 17 | Productos con traducción de categorías | 13 productos tienen una categoría que no está en la tabla de traducción: `pc_gamer` (3) y `portateis_cozinha_e_preparadores_de_alimentos` (10) | Conservar los productos y mostrar el nombre original de la categoría cuando no exista traducción |

### 4.6 Fechas

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 18 | `orders`, `order_items`, `order_reviews` | Las 8 columnas de fecha tienen un formato uniforme y se pueden convertir sin errores | Convertir el texto a fecha en la capa Plata |
| 19 | `order_items` | 4 ventas (de 3 pedidos) tienen una fecha límite de envío casi 3 años posterior a la compra (hasta abril de 2020) | Dejar esa fecha vacía cuando sea más de un año posterior a la compra |
| 20 | `orders` | 166 pedidos figuran enviados a la transportadora antes de haberse comprado. En la mayoría la diferencia es de minutos, y en un caso llega a 171 días | Conservarlos y agregar la columna de aviso `fecha_envio_inconsistente` |
| 21 | `orders` | 23 pedidos figuran entregados al cliente antes de enviarse (diferencia típica de 1.7 días, máximo de 16) | Usar la misma columna de aviso y no incluirlos al calcular el tiempo de envío |
| 22 | `orders` | Ningún pedido se aprueba ni se entrega antes de haberse comprado | No requiere acción |
| 23 | `orders` | 6 pedidos cancelados tienen fecha de entrega | Conservarlos. El tiempo de entrega se calcula solo con pedidos entregados |

Es posible que algún pedido aparezca en los puntos 20 y 21 a la vez. Eso no se verificó.

### 4.7 Valores de texto y pagos

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 24 | `orders`, `order_reviews`, `customers` | Los 8 estados de pedido, las notas (solo del 1 al 5) y los 27 estados de Brasil están bien escritos y son uniformes | Convertir la nota de la reseña a número entero |
| 25 | `order_payments` | 3 pagos tienen el método `not_defined`, con valor 0.00 y de pedidos cancelados | Conservarlos y cambiar el nombre a `no_definido` |

### 4.8 Revisión de columnas numéricas, pagos y ventas

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 26 | `order_payments` | 2 pagos con tarjeta de crédito tienen 0 cuotas (por 58.69 y 129.94, de pedidos entregados). Una compra con tarjeta tiene al menos 1 cuota, así que el 0 es un error de registro | Conservar el pago y dejar el número de cuotas vacío, sin inventar un valor |
| 27 | `order_payments` | 9 pagos tienen valor 0: los 3 `not_defined` (pedidos cancelados) y 6 con vale (`voucher`), de los cuales 4 son de pedidos entregados y 2 de pedidos en camino. Dos de ellos pertenecen al mismo pedido | Conservarlos. No afectan los ingresos porque suman 0 |
| 28 | `products` | 4 productos tienen peso 0 g, y los 4 son de la categoría `cama_mesa_banho`. Un producto no puede pesar 0 g | Conservar los productos y dejar el peso vacío, sin inventar un valor |
| 29 | `order_items`, `order_payments`, `products` | Con el método estadístico del rango intercuartílico, hay valores extremos en todas las columnas numéricas revisadas: precio (7.48%), flete (10.77%), valor del pago (7.69%), peso (13.99%), largo (4.19%), alto (7.24%) y ancho (2.77%). Son porcentajes habituales en datos con muchos productos baratos y pocos caros, y no se consideran errores | Conservar todos los valores sin modificarlos. Solo se avisa el 1% más caro con `precio_atipico` |
| 30 | `order_payments` | Los 10 pedidos con mayor pago coinciden exactamente con el total de sus productos, así que son compras reales. Uno de ellos (4,809.44) está cancelado. Los pedidos cancelados y no disponibles suman 269,735 pagados (1.68% del total) y todos los pedidos no entregados suman el 3.66% | Calcular las ventas solo con pedidos entregados (`delivered`) |
| 31 | `orders` | 1 pedido entregado (comprado el 15 de septiembre de 2016) no tiene ningún pago registrado | Conservarlo. Al unir pedidos con pagos, usar una unión que no lo descarte y dejar su monto pagado vacío |

### 4.9 Clientes, pedidos sin productos y ubicaciones

| # | Tabla | Qué se encontró | Qué se hará |
|---|---|---|---|
| 32 | `customers` | Hay 99,441 identificadores de cliente (`customer_id`), pero solo 96,096 clientes reales (`customer_unique_id`). El identificador cambia en cada pedido, así que un cliente que compra dos veces aparece dos veces | Contar los clientes con `customer_unique_id` |
| 33 | `orders` | 775 pedidos no tienen ningún producto registrado: 603 no disponibles, 164 cancelados, 5 creados, 2 facturados y 1 enviado. Ninguno está entregado | Conservarlos. No cuentan como venta. Al unir pedidos con productos, usar una unión que no los descarte |
| 34 | `customers`, `sellers` | 278 clientes (de 99,441) y 7 vendedores (de 3,095) tienen un código postal que no existe en `geolocation`. Sumados al cliente del punto 13, unos 279 clientes quedan sin ubicación | Conservarlos y dejar su ubicación vacía |
| 35 | `customers` | Las 4,119 ciudades no tienen variantes por tildes ni por mayúsculas | No requiere acción |
| 36 | `order_items`, `order_payments` | De los 98,665 pedidos que tienen productos y pagos, 98,415 (99.75%) tienen un total pagado igual al total de sus productos (precio más flete), con menos de 1 de diferencia. En 232 pedidos se pagó más (hasta 182.81) y en 17 se pagó menos (hasta 51.62). En 1 pedido la diferencia es de exactamente 1. La suma neta de las diferencias (las positivas menos las negativas) es de 2,870.39, el 0.02% de lo pagado | Usar el precio más el flete como monto facturado y el valor de los pagos como monto cobrado, siempre solo con pedidos entregados. Las diferencias no se corrigen |

## 5. Decisiones tomadas y por qué

| Decisión | Por qué |
|---|---|
| Dejar una reseña por pedido (la más reciente) | Es la opinión final del cliente y evita contar los pedidos dos veces al unir tablas |
| Conservar los pedidos con fechas incoherentes y marcarlos, en vez de borrarlos | No se sabe cuál fecha es la correcta, y borrarlos quitaría ventas reales. Con la marca, solo se excluyen de los indicadores de tiempo |
| Asignar `sin_categoria` a los 610 productos sin ficha | Todos se vendieron. Si se borraran, se perderían ventas, y una categoría vacía se vería confusa en los gráficos |
| Calcular las ventas solo con pedidos entregados | Los pedidos cancelados, no disponibles y aún en proceso tienen pagos registrados. Representan el 3.66% del dinero pagado, y sumarlos haría que las ventas salgan un 3.8% más altas |
| No modificar los valores extremos | Los de mayor valor son compras reales y pagadas. Solo se avisan, no se corrigen |
| Contar los clientes con `customer_unique_id` | El `customer_id` cambia en cada pedido, y usarlo inflaría la cantidad de clientes en 3,345 |
| Facturado = precio más flete, y cobrado = pagos | Ambas fuentes coinciden en el 99.75% de los pedidos, así que se pueden comparar sin que una distorsione a la otra |

## 6. Columnas de aviso que se crean en la capa Plata

Una columna de aviso es una columna nueva que indica sí o no. **No cambia ni borra ningún dato**: solo permite filtrar los casos dudosos cuando haga falta.

| Columna | Tabla | Qué indica |
|---|---|---|
| `entregado_sin_fecha` | `orders` | El pedido figura como entregado, pero no tiene fecha de entrega |
| `fecha_envio_inconsistente` | `orders` | Las fechas de envío o entrega están en un orden imposible |
| `precio_atipico` | `order_items` | El precio está entre el 1% más caro |

## 7. Glosario

| Término | Qué significa |
|---|---|
| Bronce | Capa con los datos tal como llegaron, sin cambios |
| Plata | Capa con los datos ya limpios y unificados |
| Oro | Capa con los datos listos para el dashboard |
| Columna de aviso | Columna nueva de sí o no que marca los casos dudosos |
| Vacío | Dato que falta en una celda |
| Identificador | Columna cuyo valor es distinto en cada fila (por ejemplo, el número de pedido) |
| Valor extremo (atípico) | Valor muy alejado de la mayoría. No siempre es un error |
| Rango intercuartílico | Método estadístico que define una zona normal con el 50% central de los datos y marca lo que queda muy fuera de ella |

## 8. Alcance y límites del análisis

Este análisis no incluyó:

- La escritura de ciudades y estados en `sellers` ni de las ciudades en `geolocation`.
- La comparación de la fecha de cada reseña con la fecha de su pedido.
- La coherencia entre el estado de un cliente y su código postal.
- La causa de las 249 diferencias entre el total pagado y el total de los productos.

Ninguna de estas revisiones afecta las reglas definidas arriba. Si en la capa Plata aparece un problema en alguno de estos puntos, se documenta como hallazgo nuevo.

## Conclusión

Se revisaron las 9 tablas y se documentaron 36 hallazgos de calidad, cada uno con la regla que se aplicará en la capa Plata.