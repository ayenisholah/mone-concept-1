# Fix: `onImgError is not defined` + `[bundle] error` banner spam

## Symptom

Loading the bundled page (e.g. `http://127.0.0.1:5500/index.html`) showed a persistent red
error banner at the bottom of the page and console noise:

```
[bundle] Uncaught ReferenceError: onImgError is not defined (http://127.0.0.1:5500/index.html:234)
[bundle] error
[bundle] error
...
```

The page itself rendered and worked correctly — the errors were boot-phase noise made
visible (and permanent) by the bundle loader's error banner.

## Root cause

`index.html` is a self-contained bundle: a loader script decodes an asset manifest, then
injects the real app (a JSON-escaped HTML template) into the live document via
`DOMParser` + `document.documentElement.replaceWith(...)`. The app uses a template-binding
runtime that compiles `{{ placeholder }}` attributes into React props — but only *after*
the swapped-in scripts execute.

In the window between `replaceWith` and compilation, the browser treats the raw template
markup as real DOM:

```html
<img src="{{ item.img }}" data-fallback="{{ item.fallback }}" onerror="{{ onImgError }}" ...>
```

1. Each `<img src="{{ item.img }}">` fetches the literal URL `{{ item.img }}`, which 404s
   and fires the image's `error` event. The loader's capture-phase `window` error listener
   logged each of these as a bare `[bundle] error` line (resource error events have no
   `.message`).
2. The failed load runs the inline handler `onerror="{{ onImgError }}"`. As JavaScript,
   `{{ onImgError }}` parses as two nested blocks containing the expression statement
   `onImgError` — evaluating an identifier that doesn't exist yet. That is the
   `ReferenceError: onImgError is not defined`. The real handler only exists later, as a
   class method the runtime compiles into a React `onError` prop.

The same transient errors exist in the non-bundled original page; the bundle only made
them visible through its error banner.

## Fix

Two changes, both in the loader script of `index.html`. The manifest and template payloads
are untouched, and no app/template code changed.

### 1. Pre-define unbound inline-handler identifiers

After parsing the template and before swapping it into the document, scan the parsed doc
for `on*` attributes that still contain `{{ … }}` and define each missing root identifier
as a `window` no-op:

```js
for (const el of doc.querySelectorAll('*')) {
  for (const a of el.attributes) {
    if (a.name.slice(0, 2) !== 'on') continue;
    const m = a.value.match(/\{\{\s*([A-Za-z_$][\w$]*)/);
    if (m && !(m[1] in window)) window[m[1]] = function () {};
  }
}
```

Transient events during boot now evaluate to a harmless no-op instead of throwing. The
attributes themselves are not modified, so the runtime still sees and compiles the
original `{{ }}` bindings — image fallback behavior (`data-fallback`) is unchanged. The
fix is generic: it covers any handler name in any future bundle, not just `onImgError`.

### 2. Ignore subresource load failures in the error sink

The banner listener now early-returns for resource `error` events, which target the
failing element and carry no `.message` — unlike real JS `ErrorEvent`s, which carry a
message and target `window`:

```js
window.addEventListener('error', function(e) {
  if (!e.message && e.target && e.target !== window) return;
  // ... append to banner as before
}, true);
```

Genuine script errors still surface in the banner.

## Verification

Rendered `http://127.0.0.1:5500/index.html` in headless Edge (`--headless=new
--dump-dom`, 15 s virtual-time budget): the resulting DOM contains no `#__bundler_err`
banner and no `onImgError` ReferenceError, and the app content (hero heading
"Next-generation aesthetic medicine.") renders correctly.
