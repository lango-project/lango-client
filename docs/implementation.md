# Implementation stages

1. **Browser foundation — implemented.** Static browser build, startup,
   opaque per-form tokens, nested/inherited values, related-form request queue,
   basic Bootstrap controls, ordered responses, navigation, and scoped template
   loading against the test bootstrap metadata.
2. **Lifecycle integration.** Align template metadata and default templates with
   Lango UI. Extend successful validation to renew expiry on the server. Define
   and implement batch Ping, tracking freshness per opaque token, waiting for
   active sends and blocking new sends while Ping runs. Add lifecycle reporting
   without request retries.
3. **Form interaction.** Add validation display/reset, replacement commands,
   complete labels and common field variants. Test disposal and pending requests
   during replacement and navigation.
4. **Control catalog and resources.** Implement the remaining declared controls,
   timer/long-poll ownership and cancellation, dialog and Toast behavior.
5. **Application delivery.** Provide a reproducible application generator,
   deep-link hosting integration, and installation/package tests against actual
   Lango Server and Lango UI applications.

All application/runtime implementations stay in Lango. Browser-specific work
belongs here; JWT claims, validation, and signing remain server-owned.
