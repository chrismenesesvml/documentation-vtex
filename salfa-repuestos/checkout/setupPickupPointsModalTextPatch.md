# setupPickupPointsModalTextPatch

IIFE declarada en `checkout-ui-custom/checkout6-custom.js` que normaliza y simplifica los textos de indisponibilidad de puntos de recogida dentro del modal de selección de *pickup points* en el paso de envío del checkout VTEX.

---

## Contexto

El modal de puntos de recogida (`pkpmodal`) muestra información sobre la disponibilidad de cada punto. Por defecto, VTEX genera textos en formato extenso/verbose:

| Casos originales | Texto normalizado |
|---|---|
| `"3 no están disponibles"` (sin acento o con acento) | `"3 no disponibles"` |
| `"1 indisponible"` | `"1 no disponible"` |

Esta función intercepta mutaciones del DOM dentro del modal para aplicar regex-based substitutions, mejorando la legibilidad sin alterar significativamente el contenido.

---

## Selectores CSS

| Selector | Propósito |
|---|---|
| `.vtex-pickup-points-modal-3-x-pkpmodal.pkpmodal` | Modal contenedor principal (raíz). |
| `.vtex-pickup-points-modal-3-x-pickupPointAvailability.pkpmodal-pickup-point-availability` | Nodo de texto con el estado de disponibilidad. Objetivo del patch. |

---

## Estado interno

| Variable | Tipo | Descripción |
|---|---|---|
| `window.__pickupPointsModalTextPatchInit` | `boolean` | Bandera global para ejecutar la IIFE solo una vez. |
| `modalObserver` | `MutationObserver \| null` | Observer activo escuchando cambios dentro del modal. |
| `rootObserver` | `MutationObserver \| null` | Observer en el `body` que detecta cuándo aparece el modal (solo en `#/shipping`). |

---

## Funciones internas

### `patchNodeText(node)`
Aplica transformaciones regex al contenido de texto de un nodo.

**Transformaciones:**
1. Detecta `"^(\d+) no están disponibles$"` (con/sin acento) → `"<número> no disponibles"`
2. Detecta `"^1 indisponible$"` → `"1 no disponible"`
3. Ignora nodos que no sean elemento (`nodeType !== 1`)

```javascript
patchNodeText(el)
// "3 no están disponibles" → "3 no disponibles"
// "1 indisponible"        → "1 no disponible"
// Otros textos no se modifican
```

### `observeModal(modal)`
Inicia la observación de mutaciones dentro de un modal ya presente.

**Lógica:**
1. Parcha todos los nodos coincidentes en el modal actual.
2. Crea un `MutationObserver` para monitorear cambios futuros (`childList: true, subtree: true`).
3. Aplica `patchNodeText` a cada cambio detectado (anti-flicker: se ejecuta apenas VTEX modifica el contenido).

### `stopModalObserver()`
Desconecta `modalObserver` si está activo y lo asigna a `null`.

### `isShippingStep() → boolean`
Verifica que `window.location.hash` comience con `#/shipping`.

### `syncModalObserver()`
Sincronización puntual: revisa si el modal está en el DOM.
- Si existe: lo observa.
- Si no existe: detiene la observación.

### `startRootObserver()`
Activa observación en `document.body` con `childList: true, subtree: false`. Solo escucha cambios directos bajo body (no nested), optimizando CPU.

Triggers `syncModalObserver()` en cada mutación detectada.

### `stopRootObserver()`
Desconecta `rootObserver` y lo asigna a `null`.

### `syncLifecycle()`
**Orquestador del ciclo de vida** basado en la ruta activa:

```
Si #/shipping activo:
  - startRootObserver()    ← escucha aparición del modal en body
  - syncModalObserver()     ← verifica si el modal existe ahora
Sino:
  - stopRootObserver()
  - stopModalObserver()
```

Se llama inicialmente al arrancar y en cada `hashchange`.

---

## Flujo de eventos

### Arranque inicial
```
1. IIFE se ejecuta
2. window.__pickupPointsModalTextPatchInit se asigna a true
3. syncLifecycle() se invoca
   ├─ Si #/shipping: startRootObserver()
   └─ Sino: stopRootObserver()
4. Listener en 'hashchange' se registra
```

### Entrada a #/shipping
```
hashchange evento
    ↓
syncLifecycle()
    ├─ startRootObserver() ← vigila body
    └─ syncModalObserver() ← verifica modal ahora
```

### Modal aparece en el DOM
```
MutationObserver (rootObserver) detecta cambios en body
    ↓
syncModalObserver()
    ├─ querySelector(MODAL_SELECTOR)
    └─ observeModal(modal) ← comienza a parchar el contenido
        ├─ patchNodeText en todos los nodos existentes
        └─ crea modalObserver para futuros cambios
```

### Usuario interactúa con el modal / VTEX actualiza disponibilidad
```
MutationObserver (modalObserver) dentro del modal detecta cambios
    ↓
patchNodeText() se aplica automáticamente
    ↓
Texto normalizado en tiempo real
```

### Salida de #/shipping
```
hashchange evento
    ↓
syncLifecycle()
    ├─ stopRootObserver() ← deja de vigilar body
    └─ stopModalObserver() ← desconecta observer del modal
```

---

## Transformaciones de texto

### Patrón 1: Múltiples indisponibles
```regex
^(\d+)\s+no\s+est[aá]n\s+disponibles$
```

**Ejemplos:**
- Entrada: `"3 no están disponibles"` → Salida: `"3 no disponibles"`
- Entrada: `"5 no estan disponibles"` (sin tilde) → Salida: `"5 no disponibles"`

### Patrón 2: Un único indisponible
```regex
^1\s+indisponible$
```

**Ejemplo:**
- Entrada: `"1 indisponible"` → Salida: `"1 no disponible"`

---

## Optimizaciones

1. **Ejecución única**: Bandera `__pickupPointsModalTextPatchInit` evita múltiples instancias.

2. **Observación bajo demanda**: `rootObserver` solo activo en `#/shipping` para minimizar CPU.

3. **Observación scoped**: `rootObserver` escucha solo cambios directos bajo `body` (`subtree: false`), no toda la jerarquía.

4. **Patching inmediato**: `modalObserver` reacciona inmediatamente a cambios dentro del modal, evitando flicker visual o desincronización.

5. **Cleanup explícito**: Observers se desconectan explícitamente al salir de `#/shipping`.

---

## Interacción con otros módulos

- **autoPickupReceiver**: Ambos atienden el paso `#/shipping`, pero operan en partes del DOM completamente diferentes. Sin conflictos.
- **setupSellerProductsInShipping**: Igualmente independiente; modifica bloques de productos, no el modal de puntos.

---

## Casos de uso

- **Estandarización de UX**: Consolida los textos de indisponibilidad bajo un formato consistente.
- **Multiidioma**: Aunque el patrón está en español, es extensible a otros idiomas.
- **Reducción de ruido visual**: El texto acortado mejora la legibilidad en pantallas pequeñas.
