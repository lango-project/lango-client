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
- `values`, `invalid`, `set`, `disable`, `enable`, `changeCaption`, `redirect`, and `setUrl`
  commands are implemented. `setUrl` changes history without fetching a view;
  `redirect` requests the target after the current response finishes.
- A sending form has a waiting indicator, blur, and `aria-busy` state.
- Forms dispose child controls and registered resources when removed. Failed
  control construction preserves the previous view and its control registry.

The Bootstrap renderer supports `form`, `container`, `row`, `col`, `text`,
`button`, `value`, `input`, `checkbox`, `switch`, `select`, `radio`, and `range`.
Fields use separate Lango classes and have visible associated labels and
descriptions. Supported value semantics:

| Control | Value |
| --- | --- |
| Input, text-like variants | String; empty input returns an empty string |
| Input `String` | Alias for the text variant; `lines > 1` creates a textarea |
| Input `Number` | JavaScript number, or null when empty/invalid |
| Input `Decimal` / `Money` | Exact text, with a decimal input hint; no float conversion |
| Checkbox | Boolean, or null while indeterminate |
| Switch | Boolean; an unset/null value becomes false |
| Range | Native numeric slider value, with min/max/steps |
| Select / Radio | The option's original JSON value, or null when unselected |

Select and Radio support structured JSON option values, compared by structure
and primitive type. Radio also supports `selectedIndex` when no `value` is
supplied, and respects each option's `enabled` flag. Readonly controls that have
no native readonly mode stay disabled even after an `enable` command.

Input `url` sends its owning form after 500 ms without another input event.
Programmatic `values` updates emit no events and create no request. Disposing
the input cancels its pending debounce and removes its listeners.

Date pickers, file/OTP controls, currency/number display formatting and advanced
decorative options remain open. Unsupported control types and input variants
fail explicitly. Decimal/Money currently preserve text rather than formatting or
parsing localized monetary input.

## Validation and form content replacement

`invalid(name, message)` resolves slash-separated names and presents plain-text
feedback. Fields receive an invalid style, `aria-invalid`, and an additional
`aria-describedby` reference while retaining their description. Before a queued
send actually starts, the sending form clears errors on its owned controls,
child forms and inherited value sources. Unrelated errors remain visible.

`set(name, control)` replaces the **contents of a named Form**, preserving that
Form's identity and token. A Form definition replaces its content directly; any
other control becomes its new content. Remaining commands, including the final
`values` command, address the new controls. Successful replacement disposes old
controls and form resources. Failed construction preserves the previous DOM,
registry, inherited names and resources, and disposes the failed candidate.
App-wide navigation listeners survive form content changes and are released
only when the application is disposed.

## Per-token keepalive

The client tracks successful response times per opaque token string. An active
form cannot postpone another form's renewal. One app timer follows the earliest
60-second deadline; equal token strings are sent once and their replacements
update every live form still holding that string. Disposed forms do not
participate, and stale results cannot overwrite a subsequently changed token.

Before Ping, current sends finish and new sends remain queued. Due tokens are
recomputed after those sends finish. Batches contain at most 64 tokens and use
the most recently refreshed live token as the Bearer credential. A batch returns
individual replacement tokens and statuses; JWT contents are never decoded.
Failed entries retain their tokens and wait until the next scheduled minute.
Business requests are never automatically replayed. Application disposal
cancels the timer.

## Timers and longpoll subscriptions

`setTimer(url, interval)` schedules one form send; `setInterval(url, interval)`
schedules repeated form sends. Delays are positive milliseconds within the
native timer range. A new command replaces the timer of the same kind and URL
in that form. Timer and interval registries are separate. `clearTimer` and
`clearInterval` are idempotent. Intervals schedule their next tick after the
previous request settles, avoiding overlapping sends and queued tick backlogs.
Each tick collects current form values and uses the current token.

Form replacement/disposal cancels timers, queued resource sends and pending
longpoll HTTP requests. Replies from stopped resources cannot modify the form.
An ordinary resource request that already reached the server may finish there;
the client cannot undo application side effects and never automatically retries
that business operation.

`startLongPoll(url, key, cursor = null)` starts one subscription per URL/key pair
in the form. Repeating a start without a cursor keeps a live subscription; an
explicit cursor replaces it. `stopLongPoll(url, key)` is idempotent. Poll requests
send only `{ key, cursor }`, not form values, with the form's Bearer token.

Held HTTP polls run outside the form send queue. Before a related ordinary send
or report, the client aborts the pending poll; Ping aborts all held polls. Polls
resume after those operations with the latest token and successfully applied
cursor. Responses from an aborted generation are discarded. Successful poll
responses enter the normal form queue, so message application cannot race form
sends or Ping. Each message's `data` is a UI response map (`view` and/or `cmds`).
Messages apply in order. The cursor advances only after successful application;
the last 256 applied message IDs also suppress recent duplicate deliveries.
A bad message stops the subscription and reports a technical failure.

Empty successful responses immediately start the next poll. Network/503/429
failures reconnect with 1–30 second exponential backoff; a numeric Retry-After
can extend the delay up to 60 seconds. Authentication and other terminal HTTP
failures stop the subscription. These reconnects repeat transport reads, not
application form operations.

On HTTP 409, the current poll stops normal reads and calls the same-origin
`fallback` URL as a normal authenticated form send. The fallback must provide
the fresh UI state and explicitly emit `startLongPoll` with its snapshot cursor.
Without that command, the subscription remains stopped. Snapshot/cursor
consistency belongs to the application. No fallback or an invalid/cross-origin
fallback stops the subscription and reports `LongPollResyncRequired` or
`LongPollResyncFailed`; a new cursor is never invented locally.

## Client error reporting

When bootstrap supplies `clientErrorUrl`, unexpected UI errors, window `error`
events and `unhandledrejection` events produce bounded technical reports:
`{ source, code, count, name? }`. Source is `ui`, `window` or `promise`. Raw
messages, stack traces, filenames, form values, channel keys and tokens are not
included. Validation feedback is not reported as an unexpected error.

Pending identical reports for the same form are coalesced up to count 100. The
app permits at most 20 queued report entries and 10 attempted reports per minute.
Reports use the ordinary form serialization/Ping gate but do not collect values,
clear validation, show a waiting indicator, or apply response views/commands.
Only their token header is accepted. Failed reports are discarded without a
Toast, recursive reporting or retry. Optional absent channels do nothing.
Global listeners survive form content replacement and close with the app.

## Custom controls

The package exports `App`, `Form`, `BootstrapRenderer`, `Control`, `Value`,
`Field`, `Input`, `Checkbox`, `Switch`, `Range`, `Select`, `Radio`,
`createTemplate`, and `cloneTemplate`. The browser entry module exports
`start()` and uses the current origin, Web platform, and Bootstrap renderer.

`renderer.registerControlType(type, factory)` registers a factory called with
`(form, props)`. It returns a control with `element`, `getValue()`,
`setValue(value)`, and `dispose()`. Commands affecting that control may also
require `setDisabled(value)` or `setCaption(value)`. Validation-aware controls
provide `setInvalid(message)` and `clearValidation()`; the exported `Control`
base supplies both and supports selecting an inner validation target. Factories must release
their event listeners and other resources in `dispose()`.

Built-in DOM templates are cloned for each control. Captions and text are
assigned through `textContent`; field values use DOM properties.

## Integration still required

- Template metadata is now supplied by Lango UI: `templates` contains reserved
  scope URLs and `defaultTemplate` optionally names the reserved default layout
  endpoint. Templates contain a named `__content__` Form. Scope matching uses
  segment boundaries and the longest matching prefix; the default handles
  unmatched pages. Template addresses themselves are not page URLs.
- The lifecycle implementation requires the updated Lango UI batch Ping and
  Lango Server token-refresh support. Publish/deploy the server first, then UI,
  and rebuild the client. Optional absent lifecycle channels remain disabled.
- The rest of the control catalog, advanced field presentation, dialogs and
  application Toast/Alert commands remain to be implemented. Unexpected send/navigation errors currently use a simple
  Toast; application errors remain application-owned.
- Query/hash history policy, SPA hosting, and packaging a
  generated application need integration work. The package currently builds one
  fixed same-origin Web/Bootstrap application, not a multi-platform generator.
