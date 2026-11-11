# Implementation stages

1. **Browser foundation — implemented.** Static browser build, startup,
   opaque per-form tokens, nested/inherited values, related-form request queue,
   basic Bootstrap controls, ordered responses, navigation, and scoped template
   loading against Lango UI bootstrap metadata.
2. **Templates and keepalive — implemented.** Scoped/default template metadata
   comes from Lango UI. Successful server validation renews expiry. Per-token
   freshness, deduplication, batch Ping and the app-wide send gate are in place.
   Application disposal cancels the keepalive timer. Client error event reporting
   remains open; business requests are never automatically retried.
3. **Form interaction — implemented for common fields.** Validation display/reset,
   Form content replacement, visible labels/descriptions, numeric/text/decimal
   inputs, textarea, checkbox/switch, structured select/radio values and range.
   Input debounce is canceled on disposal. Advanced input presentation and broader
   pending-request/navigation cases remain open.
4. **Control catalog and resources.** Implement the remaining declared controls,
   timer/long-poll ownership and cancellation, dialog and Toast behavior.
5. **Application delivery.** Provide a reproducible application generator,
   deep-link hosting integration, and installation/package tests against actual
   Lango Server and Lango UI applications.

All application/runtime implementations stay in Lango. Browser-specific work
belongs here; JWT claims, validation, and signing remain server-owned.
