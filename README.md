# Lango Client

A browser runtime for applications built with Lango UI. Application handlers run
on the server. The client renders control definitions, collects form values, sends
requests, and applies responses. Runtime, build tools, and tests are written in
Lango; JavaScript modules under `dist/` are compiler output.

This is the first implementation of the new client contract. It is not yet a
complete renderer for the Lango UI control catalog.

## Build and test

Node.js 24 or newer is required.

```sh
npm install
npm test
npm run browser:install
npm run test:browser
```

Browser installation uses `.test-output/browsers` inside this project. Browser
tests start a temporary local HTTP server and launch headless Chromium. They do
not use any other Lango Client checkout.

`npm run build` writes a static application to `dist/`, including the module
runtime, Bootstrap 5 CSS, its license, the HTML shell, and the entry module.
Configure the application's static server to serve this directory at `/`, or
copy its **entire contents**, including `.lango/`, into the public directory.
Keep POST `/` available for Lango UI startup and provide a UI endpoint at
`/index`. Build output must be served over HTTP rather than opened as a file.

GET `/` serves the unchanged HTML shell. Its entry module sends POST `/`, saves
the opaque response token, and requests POST `/index`. On a deep link, the client
requests the initial pathname instead. A host must explicitly serve the HTML
shell for those deep-link GETs; Lango Server's existing static handler returns
404 for missing files and does not yet provide an SPA fallback.

## Implemented behavior

- Startup restores the most recently saved token and sends it unchanged.
- Every form owns its current token. New child forms inherit the creating
  form's token; later responses update only the receiving form. Tokens are never
  decoded in the client. Storage errors do not prevent in-memory operation.
- Named controls contribute values, including `null` for buttons and other
  controls without values. Named child forms contribute nested maps.
  `inheritedNames` reads and writes through to the parent form.
- A form and its ancestors/descendants serialize sends. Unrelated forms may send
  concurrently. Queued sends collect values and read the current token when
  they actually start. Failed requests are not automatically repeated.
- Responses update the token first, then replace the view, then execute ordered
  commands. Partial value updates preserve unspecified controls and do not emit
  DOM change events.
- Ordinary same-origin links use client navigation. Modified clicks, downloads,
  external links, and links targeting another browsing context keep browser
  behavior. Back and forward request the page again.
- `values`, `disable`, `enable`, `changeCaption`, `redirect`, and `setUrl`
  commands are implemented. `setUrl` changes history without fetching a view;
  `redirect` requests the target after the current response finishes.
- A sending form has a waiting indicator, blur, and `aria-busy` state.
- Forms dispose child controls and registered resources when removed. Failed
  control construction preserves the previous view and its control registry.

The initial Bootstrap renderer supports `form`, `container`, `row`, `col`,
`text`, `button`, `value`, and text-like `input` variants (`text`, `email`,
`password`, `search`, `tel`, `url`). Unsupported types, variants, and commands
fail explicitly. Input labels are currently exposed through `aria-label`;
visible field labels and descriptions remain part of the renderer work.

## Custom controls

The package exports `App`, `Form`, `BootstrapRenderer`, `Control`,
`createTemplate`, and `cloneTemplate`. The browser entry module exports
`start()` and uses the current origin, Web platform, and Bootstrap renderer.

`renderer.registerControlType(type, factory)` registers a factory called with
`(form, props)`. It returns a control with `element`, `getValue()`,
`setValue(value)`, and `dispose()`. Commands affecting that control may also
require `setDisabled(value)` or `setCaption(value)`. Factories must release
their event listeners and other resources in `dispose()`.

Built-in DOM templates are cloned for each control. Captions and text are
assigned through `textContent`; field values use DOM properties.

## Integration still required

- The current Lango UI bootstrap advertises only lifecycle URLs. Scoped template
  loading is tested against an optional `templates` array of reserved URLs such
  as `['/admin']`; this metadata still needs a matching Lango UI implementation.
  A template must contain a named `__content__` form. `/admin/...` matches
  `/admin`; `/administrator/...` does not. Empty/default template metadata is
  not supported yet, and a template URL itself is not a page URL.
- Per-token freshness tracking and batch Ping are not implemented. They require
  the agreed server validation renewal and a batch lifecycle protocol. The
  current UI Ping accepts only a single Bearer token and an empty body.
- The rest of the control catalog, validation presentation, resource commands
  (timers and long polling), dialogs, toasts, and client error reporting remain
  to be implemented. Unexpected send/navigation errors currently use a simple
  Toast; application errors remain application-owned.
- Query/hash history policy, the default template, SPA hosting, and packaging a
  generated application need integration work. The package currently builds one
  fixed same-origin Web/Bootstrap application, not a multi-platform generator.
