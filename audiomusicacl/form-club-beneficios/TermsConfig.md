# TermsConfig

`TermsConfig` is a VTEX React block that centralizes the Terms and Conditions version used by checkout and hides the associated form fields from the storefront UI.

## What this block does

- Exposes the current terms version through `window.termsConfig`.
- Persists the version in `localStorage` under `termsConfigVersion`.
- Appends the version to the Club Beneficios breadcrumb when the breadcrumb exists.
- Fills the hidden checkout fields for accepted version and timestamp.
- Keeps the hidden inputs invisible even if VTEX remounts the form after the component mounts.

## Specific changes made in this folder

### First-paint hiding

The component now injects a local CSS block that hides these selectors immediately:

- `input[name="#/properties/acceptedTcVersion"]`
- `input[name="#/properties/acceptedTimestamp"]`
- `input[name="#/properties/acceptedIpAddress"]`
- Their corresponding `label.vtex-input` wrappers when present

This prevents a brief visual flash while React effects are still starting.

### DOM reinforcement

Because VTEX can render or remount the form after the block mounts, the component also uses a `MutationObserver` on `document.body`.

When matching inputs appear later, the observer re-applies the hiding logic so the fields stay hidden.

### Field population

Before submit, the component keeps writing the values required by checkout:

- `acceptedTcVersion` gets the configured terms version.
- `acceptedTimestamp` gets an ISO timestamp.

The IP field remains reserved for future use and stays hidden as well.

## File map

- `index.jsx`: main implementation for version syncing, field hiding, breadcrumb update, and checkout persistence.
- `react/TermsConfig.tsx`: top-level re-export used by VTEX block registration.

## Notes

- The block is intentionally invisible in the UI and should not render visible content.
- If VTEX changes the input names or the form structure, the selectors in `index.jsx` and the CSS selectors in this folder will need to be updated together.
- The block currently keeps the hidden fields in sync with checkout expectations without requiring user interaction.
