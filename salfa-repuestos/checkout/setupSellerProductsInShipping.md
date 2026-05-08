# setupSellerProductsInShipping

IIFE declarada en `checkout-ui-custom/checkout6-custom.js` que renderiza bloques informativos de **productos disponibles y no disponibles por seller** en el paso de envío del checkout, directamente bajo el nombre de cada vendedor en los bloques de entrega (*delivery-group-content*).

---

## Contexto

En el flujo de shipping de VTEX, cada seller aparece en un bloque de entrega separado. Esta función:

1. Agrupa los ítems del carrito por seller y disponibilidad.
2. Renderiza una lista visual de productos disponibles para retiro en tienda ("Retira estos productos en:").
3. Renderiza una lista de productos no disponibles con opción de eliminar ("X No disponible(s)").
4. Inserta estas listas inmediatamente después del texto del seller (`.vtex-omnishipping-1-x-sellerText`).
5. Detecta cambios en disponibilidad mediante `orderFormUpdated.vtex` y actualiza el DOM de forma eficiente.

---

## Mapa de sellers

Mapeo de textos visibles a IDs de seller reales (para resolver seller ID a partir del DOM):

```javascript
SELLER_TEXT_MAP = {
  'salfa repuestos': 'repuestos246',
  'salfa neumáticos y baterías': 'salinasyfabres672',
}
```

**Propósito**: VTEX renderiza el nombre del seller como texto; este mapa permite resolver el seller ID real a partir de la búsqueda textual parcial.

---

## Selectores CSS utilizados

| Selector | Propósito |
|---|---|
| `.delivery-group-content` | Contenedor del bloque de entrega individual. |
| `.vtex-omnishipping-1-x-sellerText` | Párrafo con el nombre del seller dentro del bloque. |
| `.seller-products-shipping` | Wrapper div que contiene las listas de productos (custom). |
| `.seller-products-shipping[data-seller="..."]` | Selector específico por seller ID. |
| `.seller-products-shipping__section` | Sección individual (disponibles o no disponibles). |
| `.seller-products-shipping__item` | Fila de producto individual. |
| `.seller-products-shipping__remove-section-btn` | Botón para eliminar productos no disponibles. |

---

## Estado interno

| Variable | Tipo | Descripción |
|---|---|---|
| `window.__sellerProductsShippingInit` | `boolean` | Bandera global; ejecuta IIFE solo una vez. |
| `lastRenderSignature` | `string \| null` | Hash del estado anterior. Guard contra re-renders innecesarios. |
| `latestOrderForm` | `object \| null` | Último orderForm recibido; usado para reintentos. |
| `retryRenderTimer` | `id \| null` | Timer para debounce de renderizado. |
| `isRefreshingOrderForm` | `boolean` | Bandera para evitar multiple simultáneo de `getOrderForm()`. |
| `retryCount` | `number` | Contador de reintentos; resetea cuando render es exitoso. |
| `MAX_RETRIES` | `number` | Máximo reintentos permitidos (5). |
| `RETRY_DELAYS` | `array` | Array de delays exponenciales: `[300, 600, 1000, 1500, 2000]` ms. |

---

## Estructuras de datos

### Grupo de seller
Resultado de `getSellerGroups()`:
```javascript
{
  "repuestos246": {
    sellerId: "repuestos246",
    products: [
      {
        index: 0,
        name: "Producto A",
        quantity: 1,
        imageUrl: "https://...",
        isUnavailable: false
      },
      // ...
    ]
  }
}
```

### Disponibilidad de producto
Se determina por:
1. `sellerId === '1'` → **invalid seller** → unavailable
2. `item.availability !== 'available'` → **por estado** → unavailable
3. Sino → available

---

## Funciones internas

### `isShippingStep() → boolean`
Verifica que la ruta activa sea `#/shipping`.

### `escapeHtml(str) → string`
Escapa caracteres HTML peligrosos (`&`, `<`, `>`, `"`, `'`).

### `getSellerGroups(orderForm) → object`
Agrupa items del carrito por seller y evalúa disponibilidad.

**Flujo:**
1. Iterar cada item del orderForm.
2. Agrupar por `item.seller` o `item.sellerId`.
3. Marcar `isUnavailable` si seller es '1' o availability no es 'available'.
4. Retornar objeto con estructura `{ sellerId: { products: [...] } }`.

### `buildRenderSignature(groups) → string`
Crea hash del estado actual para guard contra re-renders.

**Formato:**
```
"repuestos246::Producto A:1:0:a|Producto B:2:1:u||salinasyfabres672::Producto C:1:2:a"
```

Donde cada símbolo final es:
- `a` = available
- `u` = unavailable

### `buildPickupModeSignature(orderForm) → string`
Crea firma basada en canales de envío seleccionados.

**Formato:**
```
"0:pickup|1:delivery|2:pickup"
```

Detecta si item está en pickup-in-point o delivery.

### `buildPickupPointSignature(orderForm) → string`
Crea firma basada en direcciones de retiro seleccionadas.

Incluye addressId, addressType, postalCode, street, number.

### `buildProductsSectionHtml(title, products, unavailableSection, sellerId) → string`
Construye HTML de una sección (disponibles o no disponibles).

**HTML generado:**
```html
<div class="seller-products-shipping__section">
  <div class="seller-products-shipping__section-head">
    <p class="seller-products-shipping__section-title">Retira estos productos en:</p>
  </div>
  <ul class="seller-products-shipping__list">
    <li class="seller-products-shipping__item">
      <img class="seller-products-shipping__img" src="..." />
      <span class="seller-products-shipping__info">
        <span class="seller-products-shipping__name">Producto A</span>
      </span>
    </li>
  </ul>
  <!-- Si es unavailableSection, agregar botón de eliminar -->
</div>
```

### `buildProductListHtml(products, sellerId) → string`
Orquesta construcción de dos secciones (disponibles + no disponibles).

Retorna HTML completo con ambas secciones y botón de acción.

### `clearPrevious()`
Remueve todos los elementos `.seller-products-shipping` del DOM.

### `hasMountedSellerBlocks() → boolean`
Verifica si ya hay bloques renderizados en el DOM.

Busca `.delivery-group-content` con `.seller-products-shipping` como sibling.

### `scheduleRefresh(delay)`
Programa refresco del orderForm tras delay ms.

Si `latestOrderForm` disponible, renderiza directamente; sino, llama a `refresh()`.

### `scheduleRetry()`
Programa reintento con delays exponenciales.

Incrementa `retryCount` y usa `RETRY_DELAYS` para calcular espera.

### `resolveSellerIdFromText(text) → string|null`
Resuelve seller ID real a partir del texto visible (búsqueda en `SELLER_TEXT_MAP`).

**Ejemplo:**
```javascript
resolveSellerIdFromText("SALFA Repuestos")
// → "repuestos246" (búsqueda parcial insensitiva a mayúsculas)
```

### `render(orderForm)`
Función principal de renderizado.

**Flujo:**
1. Guard: si no en `#/shipping` → limpiar y retornar.
2. Guard: si orderForm inválido → limpiar y retornar.
3. Construir grupos de seller y firma.
4. Guard: si firma no cambió y bloques ya montados → skip.
5. Iterar `.delivery-group-content`:
   - Extraer seller ID a partir del texto visible.
   - Buscar grupo correspondiente.
   - Construir HTML del listado.
   - Insertar/actualizar elemento en DOM.
   - Limpiar elementos antiguos.
6. Si ningún bloque fue renderizado → `scheduleRetry()`.
7. Si éxito → resetear retry count.

### `refresh()`
Obtiene orderForm fresco vía API y lo renderiza.

**Guards:**
- No en `#/shipping`
- vtexjs no disponible
- Ya refrescando (flag `isRefreshingOperator`)

### `removeUnavailableFromCart(indexes, btn)`
Elimina productos del carrito por índices.

**Flujo:**
1. Validar índices válidos.
2. Mapear a payload VTEX: `{ index, quantity: 0 }`.
3. Mostrar estado "Quitando..." en botón.
4. Llamar `vtexjs.checkout.removeItems()`.
5. Al completar (ok o error) → `refresh()`.

---

## Eventos escuchados

| Evento | Comportamiento |
|---|---|
| `orderFormUpdated.vtex` | Debounce 140 ms → `render()` con nuevo orderForm. |
| `hashchange` (window) | Resetear firma y retry count. Si en `#/shipping` → `scheduleRetry()`. |
| `click` `.seller-products-shipping__remove-section-btn` | Extraer índices del button → `removeUnavailableFromCart()`. |

---

## Flujo de renderizado

```
Arranque inicial
    ↓
if (isShippingStep()) refresh()
    ↓
getOrderForm() → orderForm
    ↓
render(orderForm)
    ├─ buildRenderSignature() + buildPickupModeSignature() + buildPickupPointSignature()
    ├─ Si firma cambió o bloques no montados:
    │   ├─ getSellerGroups()
    │   ├─ Iterar .delivery-group-content
    │   ├─ Resolver seller ID
    │   ├─ buildProductListHtml()
    │   └─ Insertar/actualizar en DOM
    └─ Si éxito: resetear retryCount

orderFormUpdated.vtex dispara
    ↓
Debounce 140 ms
    ↓
render(nuevo orderForm)

Usuario hace clic en "Eliminar producto(s)"
    ↓
removeUnavailableFromCart(indexes, btn)
    ├─ btn.disabled = true
    ├─ vtexjs.checkout.removeItems(payload)
    └─ refresh() → render() con orderForm actualizado
```

---

## Validación de disponibilidad

| Condición | Resultado |
|---|---|
| `sellerId === '1'` | ❌ **unavailable** (invalid seller) |
| `item.availability === 'available'` | ✅ **available** |
| `item.availability` !== 'available' (ej: 'unavailable') | ❌ **unavailable** |
| (Sin availability field) | ✅ **available** (default) |

---

## Reintentos exponenciales

Si un render falla (ej: `.delivery-group-content` aún no en DOM):

```
Reintento 0:  300 ms
Reintento 1:  600 ms
Reintento 2: 1000 ms
Reintento 3: 1500 ms
Reintento 4: 2000 ms (máximo)
```

Tras MAX_RETRIES (5), abandona intentos.

---

## Optimizaciones

### 1. Firma triple (3D signature)
Combina:
- Grupos de seller y disponibilidad
- Modo de envío seleccionado (pickup vs delivery)
- Punto de retiro seleccionado (address)

Detecta **cualquier** cambio relevante sin re-renderizar múltiples veces.

### 2. Renderizado condicional
Checks antes de DOM update:
- `if (wrap.innerHTML !== nextHtml)` → solo actualiza si contenido realmente cambió
- `if (sellerTextEl.nextElementSibling !== wrap)` → reposiciona solo si necesario

### 3. Debounce
Espera 140 ms tras `orderFormUpdated.vtex` para acumular múltiples cambios antes de renderizar.

### 4. Limpieza selectiva
Solo remueve `.seller-products-shipping` antiguos; deja el resto del DOM intacto.

---

## Casos de error

| Escenario | Resultado |
|---|---|
| `.delivery-group-content` no en DOM | Reintenta hasta MAX_RETRIES |
| Seller no encontrado en SELLER_TEXT_MAP | Skip ese bloque (no renderiza) |
| orderForm inválido | Limpia DOM, retorna |
| fuera de `#/shipping` | Limpia DOM, retorna |
| removeItems() falla | Botón muestra estado original "Eliminar producto(s)" |

---

## Limitaciones conocidas

- **Un bloque por seller**: Si hay múltiples bloques del mismo seller (raro), solo se renderiza uno.
- **Texto visible inmutable**: Depende del texto DOM; si VTEX cambia nombres de sellers, el `SELLER_TEXT_MAP` debe actualizarse.
- **No filtra por channel**: Renderiza siempre disponibilidad general, no diferenciada por canal (pickup vs delivery).
- **Eliminación directa**: No simula carrito; realmente elimina items del orderForm.
