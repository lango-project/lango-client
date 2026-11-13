# Lango Client

A browser runtime for applications built with Lango UI. Application handlers run
on the server. The client renders control definitions, collects form values, sends
requests, and applies responses. Runtime, build tools, and tests are written in
Lango. `dist/` contains compiled application modules and separately licensed
third-party browser modules under `vendor/`.

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
runtime, Bootstrap 5 CSS, Markdown/date-picker dependencies and their licenses, the HTML shell,
and the entry module.
Configure the application's static server to serve this directory at `/`, or
copy its **entire contents**, including `.lango/` and `vendor/`, into the public
directory. The static shell includes an import map for `marked`, `dompurify` and `flatpickr`;
keep it when embedding this build in a custom HTML shell. Bundlers may instead
resolve these npm dependencies directly. No CDN is used.
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
`button`, `value`, `input`, `checkbox`, `switch`, `select`, `radio`, `range`,
`alert`, `progress`, `toast`, and `toastStack`. Dialog definitions are opened
through `showDialog`.
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

Date ranges, single-file JSON uploads, OTP and explicit numeric format settings
are implemented as described below. Advanced decorative options remain open.
Unsupported control types and input variants fail explicitly. Decimal/Money
without a format map retain their existing plain-text behavior.

## Validation and form content replacement

`invalid(name, message)` resolves slash-separated names and presents plain-text
feedback. Fields receive an invalid style, `aria-invalid`, and an additional
`aria-describedby` reference while retaining their description. Before a queued
send actually starts, the sending form clears errors on the named controls
included in its value tree, child forms and inherited value sources. Unrelated errors remain visible.

`set(name, control)` replaces the **contents of a named Form**, preserving that
Form's identity and token. A Form definition replaces its content directly; any
other control becomes its new content. Remaining commands, including the final
`values` command, address the new controls. Successful replacement disposes old
controls and form resources. Failed construction preserves the previous DOM,
registry, inherited names and resources, and disposes the failed candidate.
App-wide navigation listeners survive form content changes and are released
only when the application is disposed.

## Alerts, Toasts, Dialogs and Progress

`showAlert({ name?, message, theme?, place?, closeButton? })` accepts literal
text, a control definition or an array of text/control definitions.
The default is a dismissible alert at the start of the sending form. A placement
map uses `{ name: "email", position: "before" }`; position may also be `after`
or `inside`. Paths resolve against the sending form. A named alert replaces the
existing alert with that name; malformed replacements preserve the old alert.
`dismissAlert(name)` is idempotent for an absent alert. Closing unregisters the
control and all controls owned by its message. Named message controls belong
to the sending Form, while explicit nested Forms keep their own value/token
scopes. Replacing an alert may reuse its previous content names; collisions
with unrelated controls are rejected. A replacement cannot be anchored inside
the content it will remove. Alerts positioned in another form's subtree also close when that
subtree is replaced, without leaving a stale name in the originating form.

`showToast(toast, stack = null)` displays a Toast definition. A named ToastStack
is resolved by path; otherwise a form-owned corner region is created. Toast
header/body accept text or control definitions, including explicit Forms. A
named Toast can be addressed through its stack and content names, for example
`notifications/saved/editor/value`. Toasts themselves contribute null rather
than their internal Form values to the owner's payload. Explicit Forms inside
Toast content send their own values and may use inherited names from the owning
application form; internal presentation scopes are skipped for inheritance.

`instant: true` closes the Toast after five seconds. False or omitted leaves it
open until dismissal or owner replacement. Default position is `bottom-right`;
positions combine `top`/`middle`/`bottom` with `left`/`center`/`right`. Header/body
text is never treated as HTML. Close listeners, nested Forms and the auto-close
timer are disposed together. `Button.dismiss` closes its containing Toast or
Dialog; with a URL it dismisses only after that form send succeeds.

`showDialog(dialog, keepPrev = false)` uses a native HTML dialog with Bootstrap
content styling. Each Dialog has its own Form/token scope inherited from the
request that creates it. A root body Form supplies its controls and inherited
names; footer controls share that scope. The namespace is the Dialog's `name`,
or the root body Form's `name` when omitted, allowing a following values command
such as `editor/name`. Titles accept text or a Text definition; body/footer may
contain controls. Supported sizes are `sm`, `lg`, `xl` and `fullscreen`, with
optional dark, scrollable and backdrop-blur flags.

With `keepPrev: true`, the previous Dialog remains alive with its values,
validation, focus, resources and token. Only the active Dialog appears in the
owner's named value tree. At most the current and one previous Dialog are kept;
a subsequent retained Dialog disposes the older saved one. `swapDialog()`
exchanges those two without reconstruction. Native modality handles focus
trapping and Escape; a saved focus is restored when possible.

`showDialog(null)` closes the addressed/current Dialog and restores the saved
one when present. Programmatic closure and owner replacement do not invoke
`closeUrl`. User dismissal invokes that optional URL once using the Dialog's
values and current token, then closes after success. Repeated dismissal while
the request is pending is ignored. Failure keeps the Dialog open for an explicit
retry. New Dialog construction failures leave the active Dialog intact. Parent
view replacement or disposal closes both active and saved windows and resources.

`setProgress(name, min, max, value = null)` requires a numeric range with max
greater than min. Values are visually clamped; null is indeterminate and omits
`aria-valuenow`. `updateProgress(name, value, label = null)` preserves the range
and changes an optional application-supplied text label. The label stays visible
at zero and is exposed through `aria-valuetext`. Min/max/now ARIA values,
bar width, theme and stripe flags are updated without adding a form value. An
invalid range fails before changing the existing bar.

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

## Cards, tables and navigation

The Bootstrap renderer supports `card`, `cardGroup`, `image`, `badge`,
`toolbar`, `table`, `tab`, `nav`, `navBar` and `breadcrumb`, with their structural
item types. Text in rich slots is rendered literally; slots can also contain
control definitions or arrays of definitions. Image accepts an HTTP(S) or image
data URL, or base64 `content` with an image `mimeType` (default `image/png`).

Cards, tables, tabs and navigation do not introduce implicit Form scopes.
Controls inside them belong to the containing Form; explicit nested Forms keep
their independent values and tokens. Named non-field controls contribute null.

Tables consume `tableCol`, one optional `tableHeader`, `tableRow`, `tableCell`
and `tableDivider` definitions. Header rows contain native column headings.
`rowSpan` and `colSpan` are Cell properties; spans must be integers from 1 to
1000. Column `colSpan` uses the native column span. Numeric column widths are
pixels; strings are CSS widths. Resizable columns support pointer dragging and
Left/Right keys in 10-pixel steps, with a minimum width of 24 pixels per column.
A resizable column must end at a first-row cell boundary. Row/Cell `url` sends
the owning Form with its current token. Inner buttons, links and fields do not
also trigger a surrounding row/cell action; Enter/Space activate a focused
row/cell. All listeners are removed when the Form is replaced or disposed.

Tabs use `items` containing `tabItem` definitions. `active` selects an item by
name; omission selects the first. `variant` is `default`, `pills` or
`listGroup`. `vertical` changes both layout and arrow-key direction. Home/End
select the first/last item. `activateTabItem(name)` changes the selected panel.
Switching panels preserves their DOM, field values, token scopes and resources.
Tab and panel IDs are generated independently of application names.

NavItem/NavBarItem `url` creates a native link. Internal ordinary clicks use the
application navigation flow; external and modified clicks retain native browser
behavior. Links accept HTTP(S) URLs. A linked item cannot contain another
interactive control. Active navigation follows the longest same-origin pathname
match at a segment boundary and updates after navigation, history changes and
`setUrl`. Items without a URL use their explicit `active` flag.
When `overflow` is supplied, icon/image items can hide their text while retaining
an accessible label; `keepVisible` prevents this. Text-only items remain visible,
with scrolling when needed. Observers are disconnected on disposal. Navbar item
placement applies to its containing bar. Breadcrumbs render native links and
mark the final item as the current page.

## Menus, lists and accordion

`menu` renders a dropdown with recursive `menuItem` children. Items use `variant`
`header` or `divider`, or omit it for an action/submenu. `label`, `description`
and icons are rendered as literal text or controls; interactive controls are
not allowed inside action labels/icons. A string icon names CSS classes supplied
by the application; the runtime does not load an icon library. Description text
is exposed separately from the accessible action name.

Item `url` sends the owning Form and closes the menu. Opening a submenu never
sends a request. An item with both a URL and `subItems` gets separate action and
submenu buttons. Down/Up and Home/End move focus; Right opens a submenu, Left
returns to its parent, and Escape closes the root and restores trigger focus.
Outside pointer/focus changes close the menu. Disabling a menu closes it;
listeners on controls and the document are released with the Form.

`listGroup` wraps arbitrary control definitions from `items` in list items.
`activeIndex` is zero-based presentation metadata; null or -1 means no active
item. It does not turn the list into a field or send a selection event.
`numbered`, `flush` and `horizontal` use Bootstrap list presentation. Named
controls and explicit child Forms retain their normal value scopes.

`accordion` contains `accordionItem` definitions with `summary`, `body`,
`opened` and arbitrary JSON `value`. One item can be open by default; `multi`
allows several. More than one initially open item in a single-selection
Accordion is rejected. The named Accordion returns the open item's value or
null; in multi mode it returns an array of open item values in display order,
or an empty array. Named AccordionItems return null and remain available to
commands such as enable/disable/changeCaption.

Toggling never rebuilds bodies: fields and nested Forms retain values, tokens
and resources, including while hidden. Optional Accordion `url` sends the owning
Form once per user toggle, after the new state is applied. Setting its value
through a `values` command does not emit a toggle request. Values match by JSON
structure; invalid selections leave the previous selection intact. In single
mode null clears selection; in multi mode an array is required. Duplicate
requested values select their item once. Arrow/Home/End keys move header focus.
Summary controls must be noninteractive. `small` compacts padding and `gap`
accepts true for the default spacing, a pixel number or a CSS length.

## InputGroup and Markdown

`inputGroup` uses `controls` to place adjacent inputs/selects, text add-ons and
buttons. It is a layout control: named fields remain in their owning Form, and
explicit nested Forms keep their separate scope. A named InputGroup returns null.
Labels, descriptions and validation feedback stay attached to the original field
wrapper. CSS Grid/subgrid aligns the input row with adjacent buttons without
moving field DOM or changing event/disposal behavior.

`markdown` renders its `text` using [Marked](https://marked.js.org/) and
[DOMPurify](https://github.com/cure53/DOMPurify). Common Markdown and GFM tables,
task lists, strikethrough, links and fenced code blocks are supported. Raw HTML
is displayed literally, including inside otherwise valid Markdown. Generated
HTML passes through an explicit tag/attribute allowlist before insertion as a
sanitized DOM fragment. Script URLs, event handlers, embedded frames, forms,
SVG/MathML and application data attributes are not admitted. Code blocks remain
literal text and do not execute or load a syntax-highlighting service.

A named Markdown contributes null. `changeCaption(name, source)` updates its
Markdown source; null clears it. Invalid non-string source leaves the previous
content intact. Markdown is never used to evaluate Lango expressions or create
named Lango controls. Dependency modules and their licenses are copied to
`dist/vendor/` by the build and loaded locally through the static import map.

## Rich button captions and alert messages

Button `caption` accepts literal text, a noninteractive control definition or an
array of text/control definitions. `changeCaption(name, content)` uses the same
contract and preserves the button's action, disabled state and attributes.
Badge/Text/Image content can be used; buttons, links, fields and focusable
controls are rejected inside a button caption. Clicking caption content triggers
the containing button's action once.

Alert `message` supports interactive controls, including Buttons and explicit
Forms. Its content belongs to the sending Form; positioning the Alert in another
Form changes its lifetime rather than its request/value scope. The Alert itself
still contributes null. `changeCaption` can replace its message without removing
the close button.

Replacing either slot constructs and checks new content before committing its
DOM and name registrations. A failed definition or unrelated name collision
leaves the old slot intact and disposes partially created controls. Successful
replacement, Alert dismissal and parent disposal release old names, listeners
and child Forms. Plain strings remain literal text and avoid registry staging.
`Button` is exported as a built-in control class. The existing two-argument
`Alert(form, props)` constructor remains supported for literal text; pass the
renderer as its optional third argument when constructing rich content directly.

## ComboBox, Chip, Avatar and FormSpinner

`comboBox` is a select-only dropdown with `options` containing `value`, `control`
and optional `url`. Option content supports passive rich controls such as Card,
Text, Badge and Image; interactive/focusable controls are rejected. Content must
provide text, an explicit accessible label or image alt text. Selected values
are summarized as text on the trigger. Null-valued options introduce named
nonselectable groups. Values retain JSON types and are compared structurally.

Single selection returns its value or null. `multi = true` returns an array;
null or an empty array clears it. Duplicate requested values select their option
once. Invalid values leave the current selection intact. Down/Up and Home/End
move through options, Enter/Space select, and Escape returns to the trigger.
Outside pointer/focus changes close the popup. Readonly controls remain disabled
through `enable` commands. Popup document listeners are released on disposal.

Selecting an option sends the owning Form only when `option.url` is explicitly
present. A string value, including one that looks like a URL, remains ordinary
data. The new selection is applied before the request. Programmatic `values`
updates never invoke option actions. Multi-selection leaves the popup open.

`chip` has a fixed JSON `value` and an initial `active` Boolean. Its selection
button toggles active state and exposes `aria-pressed`. Active chips return their
configured value; inactive chips return null. Setting that value through `values`
activates it; null deactivates it. A different value is rejected. The visible
caption uses `label`, falling back to a string value. `icon` accepts CSS classes
or a passive control. Optional `button` is a separate Button definition with its
own action/disabled state, outside the selection button. Readonly/disabled apply
to selection. Neither a user toggle nor a programmatic value update sends an
implicit request; the separate Button may send one.

`avatar` displays literal text or passive controls from `content`, with optional
`online`, `offline`, `busy` or `away` status. Use an Image control for image
content. The status dot is decorative; application-supplied `aria.label` can
describe the avatar and status in the selected language. Default size is 2.5rem; `small`, `medium`, `large`, positive pixel numbers
and CSS sizes are accepted. `changeCaption` replaces content and releases its
old controls, keeping size and status. The Avatar itself returns null.

Form `spinner` accepts the name of a FormSpinner owned by that Form, or an inline
FormSpinner definition. Inline definitions can be named for later commands.
An absent setting keeps the default waiting indicator. FormSpinner `status`
is application-provided loading text; `variant` is `border` (default) or
`rotate`. `changeCaption` updates status text. It returns null and is hidden
until its Form sends an ordinary request. Ping and longpoll reads do not show it.

The spinner mounts as an overlay of its own Form, with the existing blur and
`aria-busy` behavior. It hides on success and failure; a replacement view can
supply a new spinner. Malformed spinner configuration preserves the previous
view and spinner. A Form cannot use another Form's spinner, so sibling Forms
retain independent waiting indicators. Removed controls and overlays are
released with the Form.

## Date, File, OTP and numeric formats

Input `variant: "Date"` returns a calendar range map:
`{ start: "2024-02-29", end: "2024-03-01" }`. Empty endpoints are null. `value:
null` clears both. A start without an end represents a partial range. Dates are
validated as real `YYYY-MM-DD` calendar dates without a time-zone conversion;
end cannot precede start. Invalid programmatic values preserve the previous
range. `min`/`max` use the same calendar format.

The [Flatpickr](https://flatpickr.js.org/options/) range picker supports
`multiMonth` (1–12, default 1), `firstWeekDay` (0–6, default Monday), `inline` and
button-like styling via `button`. `format` may be a Flatpickr display-format
string or a map with `display`, `locale` and optional `calendar` locale overrides.
Calendar month/day names come from Intl in the selected locale; application
strings such as month/year labels can be supplied through `calendar`.
Display formatting never changes the wire dates. Readonly/disabled blocks both
the input and calendar. Disposal destroys the picker and its event listeners.

Input `variant: "OTP"` keeps a string value, including leading zeroes.
`characters` is `numeric` (default), `alpha` or `alphanumeric`, using ASCII digits
and letters; case is preserved. `length` is 1–64, default 6. User input filters
unsupported characters before applying the logical length limit, so pasted
separators do not truncate a valid final digit. Programmatic invalid values are
rejected before mutation. Partial codes are allowed for application validation.
Unless autocomplete is disabled, the input uses `one-time-code`.

Input `variant: "File"` selects one local file and returns either null or:

```json
{ "name": "bytes.bin", "mimeType": "application/octet-stream", "size": 6, "content": "AAECA//+" }
```

`content` is base64; `size` counts the original bytes. Empty browser MIME types
use `application/octet-stream`. `maxBytes` defaults to 1 MiB (1048576 bytes) and
must be a positive safe integer. `accept` supplies the native file-chooser hint;
applications remain responsible for server-side validation. Oversized files
prevent sending. A form waits for pending file reads before collecting values,
including nested/inherited asynchronous fields. Changing/clearing/disposal
invalidates old reads and cancels their reader. Only null may be assigned through
`values`, clearing the selection; the browser never accepts a server-supplied
local file. Optional `url` is debounced after successful reading.

Base64 increases request size. Lango Server's default `bodyLimit` is 1 MiB,
which cannot hold a full 1 MiB binary file in JSON. Configure at least 2 MiB
(`bodyLimit: 2097152`) for one such file, and more if other form values require
it. This client change does not increase the server limit. Large-file streaming
and a separate upload protocol remain outside this implementation.

For Number/Decimal/Money, supplying `format` enables a formatted text input.
The map contains optional `locale` plus Intl.NumberFormat options; Money with
an explicit `currency` defaults to currency style. The client never guesses a
currency. For example:

```json
{ "locale": "tr-TR", "currency": "TRY", "maximumFractionDigits": 4 }
```

While focused, the field shows its ungrouped raw value. Outside focus it shows
the formatted value. Decimal/Money return dot-separated decimal text (empty is
an empty string); Number returns a JavaScript number or null. The editing locale's
decimal separator is accepted and normalized. Grouping/currency symbols are not
accepted as editable source text. Display rounding never changes the raw value.
Intl receives decimal strings directly, preserving large exact inputs rather
than first converting them to a JavaScript Number. Scientific decimal text is
accepted within Intl's formatting range. Programmatic invalid values preserve
the old value. No format map retains the original input behavior.

New input variants honor existing `url` debounce and programmatic-no-event
rules. Plain synchronous forms keep their original send path. Custom controls
may implement `prepareValue()` returning a Promise when value collection must
wait for asynchronous preparation.

## Custom controls

The package exports `App`, `Form`, `BootstrapRenderer`, `Control`, `Value`, `Button`,
`Field`, `Input`, `Checkbox`, `Switch`, `Range`, `Select`, `Radio`,
`ComboBox`, `Chip`, `Avatar`, `FormSpinner`, `DateInput`, `FileInput`,
`OTPInput`, `NumericInput`,
`Alert`, `Progress`, `Toast`, `ToastStack`, `Dialog`, `Menu`, `MenuItem`,
`ListGroup`, `Accordion`, `AccordionItem`, `InputGroup`, `Markdown`,
the card/table/tab/navigation
classes described above, `createTemplate`, and `cloneTemplate`. The browser entry module exports
`start()` and uses the current origin, Web platform, and Bootstrap renderer.

`renderer.registerControlType(type, factory)` registers a factory called with
`(form, props)`. It returns a control with `element`, `getValue()`,
`setValue(value)`, and `dispose()`. Commands affecting that control may also
require `setDisabled(value)` or `setCaption(value)`. Validation-aware controls
provide `setInvalid(message)` and `clearValidation()`; the exported `Control`
base supplies both and supports selecting an inner validation target. It also
records the owning `form` so positioned presentations follow its lifetime.
Factories must release
their event listeners and other resources in `dispose()`.

Built-in DOM templates are cloned for each control. Literal text uses
`textContent` or text nodes, rich slots build control definitions, and field
values use DOM properties. Markdown uses its separate sanitization path.

## Integration still required

- Template metadata is now supplied by Lango UI: `templates` contains reserved
  scope URLs and `defaultTemplate` optionally names the reserved default layout
  endpoint. Templates contain a named `__content__` Form. Scope matching uses
  segment boundaries and the longest matching prefix; the default handles
  unmatched pages. Template addresses themselves are not page URLs.
- The lifecycle implementation requires the updated Lango UI batch Ping and
  Lango Server token-refresh support. Publish/deploy the server first, then UI,
  and rebuild the client. Optional absent lifecycle channels remain disabled.
- Remaining work includes other pending controls and advanced field presentation. Unexpected send/navigation
  errors use a technical Toast; application messages remain application-owned.
- Query/hash history policy, SPA hosting, and packaging a
  generated application need integration work. The package currently builds one
  fixed same-origin Web/Bootstrap application, not a multi-platform generator.
