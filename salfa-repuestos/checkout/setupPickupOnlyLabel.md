
# setupPickupOnlyLabel

IIFE declarada en `checkout-ui-custom/checkout6-custom.js` que renderiza labels visuales en el carrito y mini-carrito para identificar productos disponibles **únicamente con retiro en tienda** (sin opción de envío a domicilio).

---

## Contexto

En VTEX, algunos ítems pueden tener configurados solo canales de entrega tipo *pickup-in-point*, sin opción de delivery convencional. Esta función:

1. Detecta cuáles items del carrito tienen `pickupOnly === true` basándose en `deliveryChannels` del orderForm.
2. Renderiza un label visual destacado en cada fila del carrito y mini-carrito.
3. Mantiene el estado estable a pesar de los re-renders continuos de VTEX.
4. Utiliza **snapshot caching** para el mini-carrito para reducir flicker.

---

## Selectores CSS utilizados

| Selector | Propósito |
|---|---|
| `.cart-template .cart-items tbody tr.product-item[data-sku]` | Filas de productos en el carrito completo. |
| `.mini-cart .cart-items > li.hproduct.item[data-sku]` | Items en el mini-carrito. |
| `.pickup-only-vtex-label` | Label row insertado después de cada fila de producto en carrito. |
| `.pickup-only-vtex-label-mini` | Label div insertado en el mini-carrito. |
| `.description` | Elemento interno del mini-carrito usado para posicionar el label. |

---

## Estado interno

| Variable | Tipo | Descripción |
|---|---|---|
| `window.__pickupOnlyLabelInit` | `boolean` | Bandera global; previene ejecución múltiple. |
| `RENDER_WAIT_MS` | `number` | Espera inicial antes de primer render (500 ms). |
| `MAX_RETRIES` | `number` | Máximo número de reintentos (8). |
| `renderTimer` | `id \| null` | Timer para debounce del renderizado principal. |
| `retryTimer` | `id \| null` | Timer para reintentos exponenciales. |
| `retryCount` | `number` | Contador de reintentos; resetea cuando render es exitoso. |
| `lastSignature` | `string \| null` | Hash del estado anterior de items. Guard para evitar re-renders innecesarios. |
| `miniObserver` | `MutationObserver \| null` | Observer activo en el mini-carrito. |
| `miniObserverTarget` | `element \| null` | Elemento específico siendo observado (`.mini-cart .cart-items`). |
| `miniObserverDebounce` | `id \| null` | Debounce timer dentro del observer del mini-carrito. |
| `miniStableSnapshot` | `object` | Map `{ domKey: { sellerId, pickupOnly } }` del último estado estable. |
| `pickupCache` | `object` | Map `{ domKey: { confirmed, streak } }` para transiciones suaves de estado. |

---

## Estructuras de datos

### itemsMeta
Array de metadatos por ítem:
```javascript
{
  domKey: "sku::ocurrencia",       // ID único de renderizado
  sellerId: "vendedorid",           // Seller del producto
  pickupOnly: true|false|null       // null = indeterminado, true/false = confirmado
}
```

### pickupCache[domKey]
Caché con estado confirmado + contador de transiciones:
```javascript
{
  confirmed: true|false|null,       // Último estado confirmado
  streak: number                    // Ciclos sin cambio para absorber ruido
}
```

### miniStableSnapshot[domKey]
Snapshot del estado estable para aplicación inmediata en mutaciones:
```javascript
{
  sellerId: "vendedorid",
  pickupOnly: true|false
}
```

---

## Funciones internas

### `buildItemsMeta(orderForm) → Array`
Construye metadatos de todos los ítems del carrito a partir del orderForm.

**Lógica:**
1. Lee `orderForm.items` y `orderForm.shippingData.logisticsInfo`.
2. Para cada ítem, busca su entrada en logisticsInfo por `itemIndex`.
3. Examina `deliveryChannels` del LogisticsInfo:
   - `hasPickup` = algún canal es `'pickup-in-point'`
   - `hasDelivery` = algún canal es `'delivery'`
4. `pickupOnly = (hasPickup && !hasDelivery) ? true : (canales.length ? false : null)`
5. Asigna `domKey` único por SKU + ocurrencia (para SKUs repetidos).

### `metaByDomKey(itemsMeta) → object`
Convierte array de itemsMeta a mapa indexado por `domKey`.

### `stablePickup(meta) → true|false|null`
**Función crítica** que suaviza transiciones de estado usando el caché.

**Flujo:**
1. Si `pickupOnly === true` → confirmar `true`, resetear streak, retornar `true`
2. Si `pickupOnly === null` → retornar estado anterior confirmado (si existe)
3. Si `pickupOnly === false`:
   - Si antes era `true` e incrementar `streak < 2` → mantener `true` (absorber 1 ciclo transitorio)
   - Sino → confirmar `false`, resetear streak, retornar `false`

**Propósito**: Evitar parpadeo cuando VTEX no tiene datos completamente disponibles aún o hay cambios transitorios rápidos.

### `buildSignature(itemsMeta) → string`
Crea hash del estado para detectar cambios reales.

Formato: `"domKey1:p|domKey2:d|domKey3:u"` donde:
- `p` = pickup-only confirmado
- `d` = delivery disponible
- `u` = unknown/indeterminado

### `prunePickupCache(itemsMeta)`
Limpia entradas obsoletas del `pickupCache` que ya no existen en el carrito.

### `createCartLabel(sellerId) → HTMLElement`
Crea fila `<tr>` con clase `.pickup-only-vtex-label` y el HTML del label.

**Contenido:**
```html
<tr class="item-unavailable pickup-only-vtex-label" data-seller-id="...">
  <td colspan="7">
    <span class="help-arrow top-arrow"></span>
    <i class="icon-warning-sign"></i>
    <span>Disponible solo con retiro en tienda</span>
  </td>
</tr>
```

### `createMiniLabel(sellerId) → HTMLElement`
Crea `<div>` para mini-carrito con clase `.pickup-only-vtex-label-mini`.

### `renderCartLabels(map) → boolean`
Renderiza labels en el carrito completo.

**Algoritmo:**
1. Iterar cada fila producto con su SKU.
2. Buscar metadatos por `domKey`.
3. Aplicar `stablePickup(meta)` para estado suave.
4. Si `null` → skip, si `false` → remover label existente, si `true` → insertar label tras fila.
5. Retornar `true` si al menos una fila fue encontrada.

### `renderMiniCartLabels(map) → boolean`
Similar a `renderCartLabels`, pero para mini-carrito.

**Particularidad:** Construye `nextSnapshot` para el siguiente estado estable.

### `renderMiniCartLabelsFromSnapshot()`
Aplica labels sin recalcular metadata; usa `miniStableSnapshot` directamente.

**Propósito**: Ejecutarse inmediatamente en mutaciones del mini-carrito para reducir flicker (50 ms más rápido que recalcular desde orderForm).

### `observeMiniCartDom()`
Inicializa MutationObserver en `.mini-cart .cart-items`.

**Lógica:**
1. Si observer ya existe y observa el mismo target → retornar.
2. Reconectar si el target cambió (VTEX remontó el DOM).
3. Escuchar `childList: true, subtree: true` para detectar cambios en items.
4. En cada mutación:
   - Aplicar snapshot inmediatamente (reduce flicker).
   - Debounce 80 ms para recalcular estado completo.

### `scheduleRetry(orderForm)`
Programa reintento exponencial si render fallido.

**Delays:** `Math.min(120 * 1.6^retryCount, 2000)` → máx 2000 ms.

### `runRender(orderForm)`
Función principal de renderizado.

**Guards:**
1. OrderForm válido y con items.
2. Carrito o mini-carrito en DOM.
3. Si firma no cambió y labels ya montados → skip.

**Flujo:**
1. Construir metadatos.
2. Limpiar caché obsoleto.
3. Construir firma.
4. Renderizar en carrito y mini-carrito.
5. Si ambos listos → resetear retry count.
6. Sino → `scheduleRetry()`.

### `schedule(orderForm)`
Debounce inicial (500 ms) antes de `runRender()` + setup del observer mini-carrito.

---

## Flujo de eventos

### Arranque
```
$(window).on('orderFormUpdated.vtex') dispara schedule()
    ↓
Espera 500 ms
    ↓
runRender(orderForm)
    ├─ Renderiza carrito
    ├─ Renderiza mini-carrito
    └─ observeMiniCartDom() ← inicia observer si falta
```

### Cambio en el orderForm
```
orderFormUpdated.vtex
    ↓
schedule(nuevoOrderForm)
    ├─ Cancela timer anterior si existe
    └─ Nuevo timer 500 ms → runRender()
```

### Cambios en mini-carrito (post-render)
```
MutationObserver dispara en .mini-cart .cart-items
    ↓
renderMiniCartLabelsFromSnapshot() ← inmediato
    ↓
Debounce 80 ms → runRender() completo si es necesario
```

### Salida de checkout / cambio de ruta
```
hashchange
    ↓
Resetear: lastSignature, retryCount, pickupCache, miniStableSnapshot
    ↓
Limpiar timers y observers
    ↓
Si aún en #/checkout → schedule()
```

---

## Mecanismo de estabilización

### El problema
- VTEX re-renderiza el DOM constantemente durante simulación de SLA y cambios de selección.
- Los `deliveryChannels` pueden no estar disponibles inmediatamente (null).
- Cambios transitorios pueden parpadear en UI.

### La solución: Triple capa de caching

1. **pickupCache**: Absorbe 1 ciclo transitorio. Si `pickupOnly` cambia de `true` a `false`, se mantiene `true` por 1 ciclo más.

2. **miniStableSnapshot**: Snapshot del último estado **confirmado** del mini-carrito. En mutaciones del DOM se aplica inmediatamente sin recalcular.

3. **lastSignature**: Guard que previene cálculos si nada cambió realmente.

---

## Optimizaciones

### 1. Firma (signature) basada en estado
Evita re-renders completamente innecesarios si los datos son idénticos.

### 2. SKU con contador de ocurrencia
Si hay 2 ítems con el mismo SKU, se diferencian con `sku::1`, `sku::2`.

### 3. Observer específico en mini-carrito
- Solo escucha cambios directos bajo `.cart-items` (`subtree: false` no sería apropiado aquí).
- Snapshot + debounce = render rápido visual + precisión eventual.

### 4. Reintentos exponenciales
No reintenta indefinidamente; máximo 8 reintentos con backoff hasta 2 segundos.

### 5. Cleanup en hashchange
Desconecta observers y limpia timers al cambiar de ruta para liberar memoria.

---

## Dependencias externas

- **VTEX orderForm API**: `window.vtexjs.checkout.getOrderForm()`, `orderFormUpdated.vtex`
- **jQuery**: `$(window).on()`
- **DOM Mutation Observer API**: Nativo del navegador

---

## Casos de uso

- **Retiro en tienda exclusivo**: Alertar al usuario que ciertos productos solo se pueden retirar, no entregar.
- **UX mejorada**: Visual claro sin requerir click adicional para expandir.
- **Consistencia**: El label persiste a través de re-renders y cambios dinámicos de disponibilidad.

---

## Notas de debugging

- Si los labels no aparecen: verificar que `.pickupOnly` en metadatos sea `true`.
- Si hay parpadeo: revisar `stablePickup()` y `pickupCache` — probablemente `deliveryChannels` está siendo actualizado constantemente.
- Si el mini-carrito no se observa: verificar que el MutationObserver se inicializó en `observeMiniCartDom()`.
- Logs disponibles: No hay console.log() activos; se pueden añadir en `runRender()` y `renderCartLabels()` para debugging.

---

## Limitaciones conocidas

- **Un label por seller**: Si hay múltiples sellers con items pickup-only, se renderiza un label genérico (no por seller individual).
- **No persiste estado de usuario**: Si el usuario añade/remueve items, el caché se resetea en `hashchange`.
- **Observer en mini-carrito**: No escucha cambios en `.cart-template` real; es solo para mini-carrito visual.
