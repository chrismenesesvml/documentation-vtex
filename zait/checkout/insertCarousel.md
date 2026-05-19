# insertCarousel: Documentacion tecnica

## Objetivo

Explicar, funcion por funcion, como opera `insertCarousel` en checkout.

## Archivo

- `checkout-ui-custom/checkout6-custom.js`

## Funcion principal: `insertCarousel`

### Responsabilidad

Controla el ciclo completo del slider de recomendaciones en checkout:

- validacion de contexto
- lectura de configuracion
- consulta de productos recomendados
- render de cards
- inicializacion de Slick
- binding de eventos de agregar al carrito

### Flujo interno

1. Verifica que la ruta sea checkout.
2. Evita crear otro slider si `.cont-carousel` ya existe.
3. Lee `/files/product-slider.json` con `fetchJsonData`.
4. Sale temprano si config no existe o `isActive` es `false`.
5. Normaliza `collectionMaxItems` en `safeMaxItems`.
6. Crea utilidades locales (`escapeHtml`, `uniqueByProductId`, etc.).
7. Obtiene productos recomendados.
8. Inserta contenedor y renderiza cards validas.
9. Inicializa Slick.
10. Asocia evento delegado para agregar al carrito.

## Utilidades locales dentro de `insertCarousel`

### `escapeHtml(value)`

Escapa caracteres especiales (`&`, `<`, `>`, `"`, `'`) para render seguro en strings HTML.

Se usa en:

- nombre de producto
- marca
- valores mostrados en `<option>`
- atributos renderizados en el card

### `uniqueByProductId(items)`

Recibe un arreglo de productos y elimina duplicados por `productId`.

Comportamiento:

- usa `Set` para registrar ids ya vistos
- descarta productos sin `productId`
- mantiene el primer elemento encontrado por id

### `getRecommendedProducts()`

Obtiene recomendaciones de accesorios para los productos actuales del carrito.

Pasos:

1. Pide orderForm con `vtexjs.checkout.getOrderForm()`.
2. Construye lista unica de `productId`.
3. Si no hay productos en carrito, retorna `[]`.
4. Para cada `productId`, consulta endpoint:
	 `/api/catalog_system/pub/products/crossselling/${accessoriesType}/${productId}`
5. Si alguna consulta falla, registra error y devuelve arreglo vacio para ese producto.
6. Aplana resultados, deduplica y limita por `safeMaxItems`.

### `getAvailableSkuOptions(productItems)`

Transforma SKUs de un producto en opciones utilizables por el selector de talla.

Reglas:

- valida que `productItems` sea arreglo
- toma `sellers[0].commertialOffer`
- excluye SKUs sin stock (`AvailableQuantity <= 0`)
- excluye SKUs sin `itemId`
- devuelve objetos `{ itemId, label }`

Label usado:

- `Talla`
- `name`
- fallback `'Talla unica'`

### `resolveCarouselPrice(num)`

Normaliza y formatea precio para locale `es-CL` sin decimales.

- trunca valor con `Math.trunc`
- usa `Intl.NumberFormat` reutilizado (`priceFormatter`)
- fallback a `0` si valor no es numerico

### `bindAddToCartEvents()`

Asocia el click en botones `.shop` usando event delegation sobre el contenedor del slider.

Detalles:

- limpia handler previo con namespace `click.carouselAddToCart`
- previene doble click con estado `is-loading`
- muestra estado visual de carga en el boton
- obtiene SKU desde:
	- opcion seleccionada en `.talla-select`
	- fallback `data-default-sku`
- ejecuta `vtexjs.checkout.addToCart([item], null, 1)`
- muestra toast de exito o error
- restaura estado original del boton al finalizar

## Inicializacion del slider (Slick)

`insertCarousel` configura Slick con:

- `dots: true`
- `infinite: false`
- autoplay
- flechas personalizadas con Font Awesome
- breakpoints para desktop/tablet/mobile

Si no hay cards validas para renderizar, elimina `.cont-carousel` y no inicializa Slick.

## Configuracion esperada en `product-slider.json`

Campos usados actualmente:

- `isActive`: habilita/deshabilita el slider
- `accessoriesType`: tipo de cross-selling (default `accessories`)
- `collectionMaxItems`: maximo de items (default `15`)

## Resultado funcional esperado

- En checkout, el slider muestra accesorios relacionados a productos del carrito.
- Si no hay accesorios validos, no se muestra slider.
- Cada card agrega su SKU correcto al carrito.
