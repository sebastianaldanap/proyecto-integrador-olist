# Revisión de la calidad de los datos (EDA): Olist

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
- Los problemas son pocos en cantidad, salvo en dos tablas: `geolocation` (26% de las filas están repetidas) y `order_reviews` (547 pedidos tienen más de una reseña).
- **Ninguna regla borra pedidos ni ventas.** Cuando un dato es dudoso, se conserva y se marca con una columna de aviso.

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
| 1 | Todas | Ninguna fila tuvo problemas de lectura | No requiere acción |

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
| 14 | `order_items` | Los precios son válidos (no hay ceros ni negativos). El precio típico es de 74.99, el 1% más caro supera 889.99 y el máximo es 6,735. Los 10 más altos son razonables, salvo uno de la categoría `utilidades_domesticas` (6,735), que resulta dudoso y no se pudo confirmar | Conservar todos y agregar la columna de aviso `precio_atipico` para precios sobre 889.99 |
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

## 5. Decisiones tomadas y por qué

| Decisión | Por qué |
|---|---|
| Dejar una reseña por pedido (la más reciente) | Es la opinión final del cliente y evita contar los pedidos dos veces al unir tablas |
| Conservar los pedidos con fechas incoherentes y marcarlos, en vez de borrarlos | No se sabe cuál fecha es la correcta, y borrarlos quitaría ventas reales. Con la marca, solo se excluyen de los indicadores de tiempo |
| Asignar `sin_categoria` a los 610 productos sin ficha | Todos se vendieron. Si se borraran, se perderían ventas, y una categoría vacía se vería confusa en los gráficos |

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
