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
5. **Control catalog.** Implement the remaining controls, advanced fields,
   Alert/Toast/Dialog commands, progress updates, tabs and navigation controls.
6. **Application delivery.** Provide a reproducible application generator,
   deep-link hosting integration, and installation/package tests against actual
   Lango Server and Lango UI applications.

All application/runtime implementations stay in Lango. Browser-specific work
belongs here; JWT claims, validation, and signing remain server-owned.
