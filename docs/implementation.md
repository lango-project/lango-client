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


Table column metadata accepts `caption` and `width`. Widths are integer units from 1 to 12;
for example, 4 requests one third of the table width and 6 requests half.
`-1` leaves the width unset for automatic browser layout. If at least one column
has a caption and the table has no explicit `tableHeader`, the renderer creates
one header row using the column captions and spans. Explicit headers take
precedence over column captions.
