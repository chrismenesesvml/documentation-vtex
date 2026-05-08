
# autoPickupReceiver

IIFE declarada en `checkout-ui-custom/checkout6-custom.js` que rellena automáticamente el campo **"Nombre del receptor"** del flujo de retiro en tienda (*pickup-in-point*) con el nombre completo del cliente autenticado, evitando que el usuario tenga que escribirlo manualmente.

---

## Contexto

En el checkout de VTEX, cuando se selecciona un punto de retiro, aparece un campo editable con el nombre del receptor. Por defecto ese campo puede aparecer vacío o con un valor incorrecto. Esta función lo parchea directamente en el DOM en cuanto detecta que:

1. El paso activo es `#/shipping`.
2. Al menos un ítem del carrito tiene `selectedDeliveryChannel === 'pickup-in-point'`.
3. Existe un nombre de cliente en `orderForm.clientProfileData`.

La persistencia vía API queda **deshabilitada intencionalmente** para evitar errores `403 Forbidden` en cuentas con permisos restringidos. Solo se parchea el DOM.

---

## Estado interno

| Variable | Tipo | Descripción |
|---|---|---|
| `_lastAutoSet` | `string\|null` | Último nombre escrito por la función. Permite distinguir ediciones del sistema vs. del usuario. |
| `_userChanged` | `boolean` | `true` cuando el usuario editó el campo manualmente. Bloquea futuros auto-sets. Se resetea al salir y volver a `#/shipping`. |
| `_patching` | `boolean` | Bandera anti-reentrada activa mientras se manipula el DOM del nodo receptor. |
| `_lastObservedState` | `object` | Snapshot `{ pickup, receiver, client }` del último `orderFormUpdated` procesado. Sirve de guard para no re-renderizar sin cambios reales. |
| `_receiverObserver` | `MutationObserver\|null` | Observer activo sobre el nodo `.shp-pickup-receiver__name`. Se reconecta si VTEX desmonta y remonta el componente. |
| `_domRetryCount` / `_domRetryTimer` | `number / id` | Contador y timer para reintentos cuando el nodo DOM aún no existe. Máximo **8** reintentos cada 120 ms. |

---

## Selectores CSS utilizados

| Selector | Uso |
|---|---|
| `.vtex-omnishipping-1-x-name.shp-pickup-receiver__name, .shp-pickup-receiver__name` | Nodo de texto del nombre del receptor. |
| `.shp-pickup-receiver__text` | Contenedor del input del receptor (detección de edición manual). |
| `.vtex-omnishipping-1-x-textBox` | Contenedor alternativo del input (detección de edición manual). |

---

## Funciones internas

### `getClientFullName() → string`
Lee `orderForm.clientProfileData.firstName` y `lastName`, los une con espacio y devuelve el nombre completo. Devuelve `''` si el orderForm no está disponible.

### `getCurrentReceiverFromOF() → string`
Lee `orderForm.shippingData.address.receiverName`. Devuelve `''` si no existe.

### `getAutoPickupState() → { pickup, receiver, client }`
Snapshot del estado relevante del orderForm:
- `pickup`: `true` si algún ítem tiene `selectedDeliveryChannel === 'pickup-in-point'`.
- `receiver`: nombre actual del receptor en el orderForm.
- `client`: nombre completo del cliente.

### `isShippingStep() → boolean`
Comprueba `window.location.hash` contra el prefijo `#/shipping`.

### `hasRelevantStateChange() → boolean`
Compara el estado actual con `_lastObservedState`. Si cambió cualquier propiedad (`pickup`, `receiver` o `client`), actualiza el snapshot y devuelve `true`. Evita reprocesamiento innecesario en cada `orderFormUpdated`.

### `_cleanReceiverNode(node, name)`
Manipula directamente los text nodes hijos del nodo receptor para escribir `"<nombre> - "`. Consolida múltiples text nodes en uno solo. Protegida con la bandera `_patching` para evitar bucles al disparar mutaciones del observer.

### `patchReceiverDOM(name) → boolean`
Orquesta el patch del DOM:
1. Llama a `_cleanReceiverNode`.
2. Conecta (o reconecta) `_receiverObserver` al nodo para mantener el valor ante re-renders de VTEX.
3. Devuelve `false` si el nodo aún no existe en el DOM.

### `_stopReceiverObserver()`
Desconecta el observer, limpia el debounce y cancela el timer de reintentos DOM. Se llama al salir del paso shipping o cuando no hay pickup seleccionado.

### `_scheduleDomRetry()`
Programa un reintento de `tryAutoSet()` en 120 ms. Máximo `DOM_RETRY_MAX` (8) reintentos.

### `tryAutoSet()`
Función principal de lógica. Flujo:

```
isShippingStep()?  ──No──► return
_userChanged?      ──Sí──► return
state.pickup?      ──No──► _stopReceiverObserver(); return
clientName?        ──No──► return
¿usuario editó?    ──Sí──► _userChanged = true; return
patchReceiverDOM() ──No──► _scheduleDomRetry(); return
¿ya persistido?    ──Sí──► return
_lastAutoSet = clientName
```

---

## Eventos escuchados

| Evento | Comportamiento |
|---|---|
| `hashchange` (window) | Al entrar a `#/shipping`: resetea `_userChanged` y programa `tryAutoSet` con 400 ms de espera. Al salir: detiene el observer y resetea `_userChanged` si el valor en el orderForm coincide con el último auto-set. |
| `orderFormUpdated.vtex` (jQuery) | Si hubo cambio relevante de estado y se está en `#/shipping`, llama `tryAutoSet` con 60 ms de debounce. |
| `change` (document, capture) | Detecta ediciones manuales del usuario en el input del receptor y activa `_userChanged`. |

---

## Arranque inicial

Si la página carga directamente en `#/shipping` (p. ej. recarga de página en paso 2), se ejecuta `tryAutoSet()` con un retraso de **600 ms** para esperar que omnishipping termine de renderizar.

---

## Diagrama de flujo simplificado

```
Página carga / hashchange → #/shipping
         │
         ▼ (600 ms / 400 ms)
     tryAutoSet()
         │
    ┌────┴─────────────────────────┐
    │ pickup seleccionado? NO      │──► _stopReceiverObserver()
    └──────────────────────────────┘
         │ SÍ
    ┌────┴─────────────────────────┐
    │ usuario editó manualmente?   │──► _userChanged = true → salir
    └──────────────────────────────┘
         │ NO
    ┌────┴─────────────────────────┐
    │ patchReceiverDOM(clientName) │──► nodo no existe → _scheduleDomRetry()
    └──────────────────────────────┘
         │ OK
    _lastAutoSet = clientName
         │
    MutationObserver mantiene el valor
    ante re-renders de VTEX omnishipping
```

---

## Limitaciones conocidas

- **No persiste en la API**: El `receiverName` no se guarda en `orderForm.shippingData.address` vía API. Si el usuario refresca la página en un paso posterior, el campo puede aparecer vacío nuevamente hasta que `tryAutoSet` vuelva a ejecutarse.
- **Un solo receptor para todos los ítems**: La función rellena el campo global de receptor; no diferencia por grupo de entrega o seller.
