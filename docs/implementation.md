# Implementation stages

1. **Browser foundation — implemented.** Static browser build, startup,
   opaque per-form tokens, nested/inherited values, related-form request queue,
   basic Bootstrap controls, ordered responses, navigation, and scoped template
   loading against Lango UI bootstrap metadata.
2. **Templates and keepalive — implemented.** Scoped/default template metadata
   comes from Lango UI. Successful server validation renews expiry. Per-token
   freshness, deduplication, batch Ping and the app-wide send gate are in place.
   Application disposal cancels the keepalive timer. Business requests are never
   automatically retried.
3. **Form interaction — implemented for common fields.** Validation display/reset,
   Form content replacement, visible labels/descriptions, numeric/text/decimal
   inputs, textarea, checkbox/switch, structured select/radio values and range.
   Input debounce is canceled on disposal. Advanced input presentation and broader
   pending-request/navigation cases remain open.
4. **Background resources and error reporting — implemented.** Form-owned one-shot
   timers and non-overlapping intervals, cancellable longpoll with serialized
   delivery, cursor/deduplication/backoff/resync, and bounded technical error
   reports. Polls yield to ordinary sends and Ping; report failures cannot recurse.
5. **Presentation commands — implemented.** Named/positioned Alerts, Toasts and
   ToastStacks, timed Toast disposal, native Dialog keep/swap/close lifecycle,
   independent dialog token/value scopes and accessible Progress updates.
6. **Control catalog — partially implemented.** Cards/images/badges/toolbars,
   native tables with row/cell actions and column resizing, persistent tabs,
   URL-aware navigation, breadcrumbs, recursive menus, ListGroup and value-bearing
   Accordion, field-preserving InputGroup and sanitized Markdown with local
   dependency assets. Rich Button captions and Alert messages include owned
   content cleanup and rollback. ComboBox, selectable Chip, Avatar and form-owned
   FormSpinner, Date range picker, single JSON/base64 File, OTP and explicit
   numeric/currency formatting are implemented. Decorative/input refinements
   remain open.
7. **Application delivery — first Web/Bootstrap flow implemented.** The UI package
   creates a server application; the Client CLI writes static browser assets.
   Lango Server supports opt-in extensionless HTML fallback and public compiler
   runtime files. Packed installation, generated handler tests/builds, stable
   signing keys, compiler-free startup and actual Chromium form/deep-link flows
   are verified by the starter delivery test. Additional platforms and query/hash
   navigation policy remain open.

All application/runtime implementations stay in Lango. Browser-specific work
belongs here; JWT claims, validation, and signing remain server-owned.


Table column metadata accepts `caption` and `width`. Positive widths are character
units (`ch`), including fractional values.
`-1` leaves the width unset for automatic browser layout. If at least one column
has a caption and the table has no explicit `tableHeader`, the renderer creates
one header row using the column captions and spans. Explicit headers take
precedence over column captions.


Common Control dimensions use `ch` for `width` and `lh` for `height`; omitted,
null or `-1` means automatic sizing. Widths are bounded by the available space.
Row uses wrapping flex layout with `small`, `normal`, or `large` gaps. On wide
viewports, Col uses its character width as its basis, with optional boolean
`grow` (default true only when no explicit width is present). Below 48rem, Col
children stack at full available width. At exactly 48rem, wide layout applies.
The CSS and request classification use the same media query.

The browser host samples `(min-width: 48rem)` for every HTTP request, including
startup, sends, Ping, reports and longpoll exchanges, and sends `X-Lango-Screen`
as `wide` or `narrow`. Resizing changes layout through CSS without issuing an
HTTP request or recreating the current view. The next request reports the new
classification. Headless/custom App hosts without a screen callback default to
wide. No dimensions or mode are persisted in tokens or storage.


If startup POST / rejects a saved opaque token with HTTP 401, the client removes
only `lango.token` from localStorage and stops. It does not retry startup or issue
page requests. A descriptive technical console error tells the user to reload;
the next explicit reload starts without a saved token. Other startup failures
and ordinary form/Ping/poll failures do not trigger this cleanup or retry.
The client never decodes JWT claims. If storage cannot be cleared, the diagnostic
reports that fact instead of claiming a new session was started.


The `container` wire type is a standalone layout group. Its optional boolean
`fluid` defaults to false and chooses Bootstrap `.container` or `.container-fluid`.
It registers children in the current Form and creates no separate value/token
scope. Other controls do not inherit from the layout Container in the UI API.
The HTML `#app` mount has no `.container` class; applications select their own
width policy. The generated UI template supplies a bounded layout Container.
The existing 48rem Row/Col and screen-header boundary is unchanged.


Table data binding is implemented: optional row templates, safe text fallback,
independent cell control scopes, none/cells/rows/mixed value collection, separate
row metadata, asynchronous field preparation, and incremental row/cell commands.
Full replacement stages rows before publishing names; errors preserve the previous
rows. Replaced rows release nested Forms and controls. Chromium tests cover the
value and lifecycle contracts; the packed starter sends template-edited row values
through real UI middleware without a compiler installed at runtime.
