# VYRE Audio 2.1.1 — Community Plugin and Engine SDK

This is the release contract for external developers. Plugins run untrusted code in isolated renderers, not in the native host or DSP process. Examples below describe data shapes and are not a complete calibration engine or a distributable plugin.

# VYRE Universal Extension API

The universal API exposes `vyre.crossover` and `vyre.subwoofer` with operation `replace`, `vyre.crossover` with `constrain`, `vyre.speakerEq` with `transform`, `vyre.targetCurve`, `vyre.ui`, `vyre.settings` and `vyre.measurements` with operation `extend`, and `vyre.calibration` with `observe`; all use contract version 1. They use the existing native integrations, sandbox and mandatory safety checks. Reading existing measurements through `api.measurements` requires the separate `measurements:read` permission; supplying measurements through `vyre.measurements` does not grant that read permission. Unsupported points, operations, versions and permissions fail closed.

Declare the extension in the existing plugin manifest (other required metadata omitted):

```json
{
  "capabilities": ["algorithm-replacement"],
  "permissions": [],
  "extensions": [
    {"extensionId":"my-crossover", "point":"vyre.crossover", "operation":"replace", "contractVersion":1, "permissions":[]}
  ]
}
```

Ownership comes from the validated plugin ID and is bound to the existing manifest/code digest. `extensionId` is unique within the owning plugin. The plugin entry and values passed to it are exactly the existing [crossover contract](ALGORITHM-REPLACEMENT-SDK.md): in particular `api.stage` remains `crossover-v1`, `api.mode` is `replace`, and the input and output retain version 1. The entry point is sandboxed as before, with the existing timeout, cancellation, memory and digest checks.

For the subwoofer stage, use `point:"vyre.subwoofer"` with `operation:"replace"` and `contractVersion:1`; the existing `subwoofer-optimization-v1` contract and topology restrictions apply. Both universal points may be declared in the same manifest with distinct extension IDs.

## Native crossover constraints

Declare `{"extensionId":"limits","point":"vyre.crossover","operation":"constrain","contractVersion":1,"permissions":[]}` under the existing `algorithm-replacement` capability. The enable dialog and plugin settings identify that this extension can affect native crossover calculations. The sandbox receives `api.mode:'constrain'`, `api.stage:'crossover-constraints-v1'`, and `input` containing `apiVersion:1`, `stage`, `receiverId`, the sorted, already supported `stagesHz`, measured speaker `channels`, and `allow40HzCrossover`. No raw measurements are included; the separate `measurements:read` permission still applies.

Return `{apiVersion:1, minHz?, maxHz?, allowedHz?}`. Bounds must be finite 40–250 Hz, with minimum no greater than maximum. An optional nonempty `allowedHz` must consist solely of unique stages in `input.stagesHz`. Omitting a field imposes no corresponding restriction. Constraints currently apply to the complete native speaker crossover stage, not to individual channels.

Enabled owners execute in plugin/extension ID order with digest-bound, per-calibration settings and storage snapshots. Each receives the same original candidate list. The host intersects all returned restrictions; order and installation order do not confer priority. Empty intersections, unsupported stages, malformed outputs, changed or disabled owners, timeouts and measured-speaker safety conflicts abort the calibration without unrestricted fallback. The native scorer, pairing and subsequent diagnostics continue to choose and validate the result. The final native channel choices and ADY crossover codes are checked against the constrained set. Manual user crossover overrides and electrical-only export are rejected while a crossover constraint is active. An active crossover replacement conflicts explicitly with crossover constraints; a separately registered subwoofer replacement remains independent. Engine API 2 is unchanged.

## Native speaker EQ transforms

Declare `{"extensionId":"eq","point":"vyre.speakerEq","operation":"transform","contractVersion":1,"permissions":[]}` under the existing `algorithm-replacement` capability. The plugin manager and enable dialog disclose its native speaker EQ effect. In the sandbox, `api.mode` is `transform`, `api.stage` is `speaker-eq-transform-v1`, and `input` contains `apiVersion:1`, `stage`, `frequenciesHz`, `maxBoostDb`, `policy`, and `channels:[{id,correctionDb,minDb,nativeMaxDb,boostLimitDb,maxDb,correctionLimitHz}]`. The 1053 frequencies are the native 10 Hz × 2^(i/96) grid. Each curve is the current automatic speaker correction before manual PEQ; it never includes a subwoofer. Raw measurements require the separate `measurements:read` permission.

Return a **complete replacement curve**, `{apiVersion:1,channels:[{id,correctionDb}]}`, for every supplied channel. Policy `native-bounded-lf-boost-and-cut-transform` permits bounded attenuation from 20 Hz to the active channel correction limit. Between 20 and 250 Hz it also permits raising an original negative native correction toward zero and, if the native boost guards allow it, into positive gain. A neutral or positive native correction may likewise be raised there. Outside that independently justified band, additional positive gain is not released; native cuts above 250 Hz remain protected.

The host derives `boostLimitDb` for each bin from the actual native per-frequency boost limit and existing null protection. The native selected +5/+10 dB automatic-EQ cap still applies, and below 40 Hz the native +2 dB cap applies. When the measured low-frequency or deep-null guard blocks boost, `boostLimitDb` is zero. This information is derived before the sandbox runs, from the same native smoothed speaker measurements and limits. The released band is conservative because native speaker regularization does not couple bins at or below 250 Hz to the 2.5–9 kHz sibilance guard; above 250 Hz the solver does not provide enough per-bin provenance to safely distinguish all protective contributions.

For each bin `minDb` is the native cut bound, `nativeMaxDb` is the original native correction, `correctionDb` is the preceding validated correction, and `maxDb` is the maximum accepted result in that step. Plugins may reverse earlier attenuation inside an eligible LF bin but cannot expand its fixed native boost limit: the bound applies to the accumulated output, not separately to each plugin. If a boost guard blocks a bin, an eligible native cut can still be reduced only to 0 dB. Elsewhere the original and preceding correction limit increases. Below 20 Hz and above the active correction limit values stay unchanged. Unknown or duplicate channels, additional fields, invalid grids, non-finite values and results outside the envelope abort without clipping. The contract remains `contractVersion:1`.

The cap applies to automatic EQ; manual PEQ is added separately. Native export validation rejects combined automatic and manual EQ above +24 dB and automatic EQ above the selected cap. The existing later speaker normalization can transfer gain between curve and receiver trim while verifying their sum; it is not permission to exceed the pre-normalization budget. Passing these software checks does not establish hardware or acoustic clipping freedom. See `SPEAKER-EQ-FINAL-ACCEPTANCE.md` for tested configurations and measurement coverage.

Enabled transformations run in plugin-ID and extension-ID order. Each receives the previous validated output, in a detached sandbox copy, while every step is checked against the original native safety envelope. Input and output are bounded to 2 MiB; the existing timeout, memory watchdog, cancellation, digest binding and permission checks apply. A per-calibration session snapshots all active constraint and transform owners, settings and storage, including a shared snapshot when the same plugin also owns an independent replacement stage. Deactivation, changed digests or owner changes invalidate the run. The transform is applied before native level analysis, crossover selection, diagnostics, final curves and normal ADY export; normal native headroom and export validation remain active. Changes in the speaker bands used by the native level and bass-integration calculations can consequently change native speaker/subwoofer levels, subwoofer distance and crossover choices; these are recalculated by the native engine rather than controlled directly by the transform. Subwoofer EQ data is never passed to the transform. Electrical-only export rejects an active speaker transform. Independent Engine API 2 engines never invoke the native transform. No speaker-EQ replacement stage is currently registered; independent crossover and subwoofer replacements remain available under their own contracts.

Legacy `replacements` declarations continue to work. A single plugin can declare several supported extensions and can combine a universal crossover declaration with the legacy subwoofer stage. Legacy and universal declarations resolve into one registry: two enabled owners for the same stage, or duplicate extension identities, abort instead of choosing a winner or calling a plugin twice. A plugin failure or an invalid response aborts without native fallback.

## Target curve extensions

Declare `{"extensionId":"targets","point":"vyre.targetCurve","operation":"extend","contractVersion":1,"permissions":[]}` in the existing `extensions` array. The plugin retains `algorithm-replacement` as the existing sandbox capability; no system permissions are granted. On request `api.mode` is `extend`, `api.stage` is `target-curve-content-v1`, and the input carries only the stage and version. Return `{apiVersion:1,curves:[{id,name,description,points}]}`. `description` is optional and `points` are `[frequencyHz, gainDb]` pairs. IDs use lowercase ASCII letters, digits and hyphens. Points must be finite, unique in frequency, within the existing curve domain, and each curve has at least two points.

Multiple plugins and declarations may contribute curves. Each output is validated independently and sorted by plugin, extension and curve ID. The existing sandbox result limit is 32,768 serialized characters; the adapter additionally limits the total to 4096 points per result. The manifest retains its existing 16-declaration and size limits. IDs in the Studio are namespaced by plugin, extension, curve and approved digest, and cannot replace native preset IDs.

Enabled extensions are refreshed by the plugin manager, on detected changes in the plugin folder, and before calibration. The Studio lists them as plugin curves; global and group selections use their actual points. If a selected curve becomes unavailable, its editor copy is cleared. On disable, removal, invalid output or digest change, a selected old curve stays visibly unavailable; calibration cannot silently fall back to another target. Reapproval of an updated plugin produces new digest-bound IDs, requiring a fresh selection. Plugin code always runs in the existing isolated sandbox and never receives host DOM access.

## UI extensions

Declare `{"extensionId":"my-ui","point":"vyre.ui","operation":"extend","contractVersion":1,"permissions":[]}` in `extensions`. Each enabled declaration returns `{apiVersion:1,tabs:[{id,title,description,view}]}`; `description` is optional and `view` is `{apiVersion:1,components:[...]}`. IDs are lowercase ASCII letters, digits and hyphens. Tabs are namespaced by plugin, extension, tab and approved digest. Multiple owners and multiple tabs are supported; entries are ordered by their namespaced IDs and never replace native pages.

The shared, declarative host renderer accepts `heading`, `paragraph`, `metric`, `button`, `chart`, `input`, `select`, `checkbox`, `slider`, and `table`. Each component uses bounded, validated text and values; HTML, CSS, URLs and scripts are not accepted as component fields. Text is rendered as text, not HTML. Input controls have unique IDs; buttons declare an action. A button invokes the owning plugin in the existing sandbox with `input.tabId`, `input.action` and `input.values`, and expects `{apiVersion:1,view:{apiVersion:1,components:[...]}}`. Results replace only that plugin tab's view. A declaration may return at most 64 tabs, component count is limited to 30 per view, and the existing 32,768-character serialized sandbox result limit applies.

Component fields are:

- `heading`, `paragraph`: `type`, `text`; `metric`: additionally `value` (string).
- `button`: `type`, `text`, `action` (lowercase letters, digits, `_` or `-`).
- `input`: `type`, `id`, `text`, `value` (string); `select`: the same plus `options` (unique strings).
- `checkbox`: `type`, `id`, `text`, `value` (boolean); `slider`: `type`, `id`, `text`, `value`, `min`, `max`, `step` (finite numbers).
- `table`: `type`, `text`, `headers` (strings), `rows` (equally sized arrays of strings); `chart`: `type`, `text`, `points` (finite `[x,y]` pairs).

Input IDs are unique within a view and use lowercase letters, digits, `_` or `-`; reserved object keys are rejected. On a button action, `input.values` contains the currently validated controls keyed by their IDs. The action result supplies the next complete `view`; it does not mutate host UI directly.

The host serializes extend calls through the existing isolated execution path, with at most 32 pending calls. Disabling or changing an owner cancels its running action and removes its digest-bound tabs; updated code needs the existing reapproval. UI actions do not receive native engine objects or host DOM access. Plugin tab view state lasts while the registration remains active; persistent declarative settings are provided by `vyre.settings` and can also appear in native VYRE areas as described below. Actions are unavailable while a native calibration is running. There is no dedicated per-action cancel control; the existing cancellation mechanism and plugin deactivation terminate active execution.

## Plugin settings and storage

Declare settings as validated manifest data on the existing universal dispatcher, for example:

```json
{"extensionId":"prefs","point":"vyre.settings","operation":"extend","contractVersion":1,"permissions":[],"settings":[{"key":"enabled","label":"Enabled","type":"boolean","default":true}]}
```

Supported types are `boolean`, `string`, `number`, `select` and `slider`. Numeric fields may provide `min`, `max`, and `step`; selects provide unique string `options`; `group` and `description` supply headings and help. Each declaration supports up to 32 unique keys and the existing manifest permits at most 16 extension declarations. Reserved object keys are rejected. The manifest schema is declarative and validated by the host; it is not executable code. Active settings appear under the owning plugin in the existing Settings UI. In sandbox actions, `api.extensionSettings[extensionId][key]` contains that plugin's validated value snapshot; the existing `api.settings` continues to provide legacy manifest settings.

A `vyre.settings` field of type `boolean`, `slider`, or `select` may additionally declare a `placement`: `calibration` (Calibration Settings), `advanced` (Advanced Options), `target` (Target Curve Studio), or `app` (App Settings / General). The host renders the existing setting at that fixed location with native controls and the active theme. For example:

```json
{"extensionId":"prefs","point":"vyre.settings","operation":"extend","contractVersion":1,"permissions":[],"settings":[{"key":"enabled","label":"Enable processing","type":"boolean","default":false,"placement":"advanced"},{"key":"strength","label":"Strength","type":"slider","min":0,"max":10,"step":1,"default":5,"placement":"calibration"},{"key":"mode","label":"Mode","type":"select","options":["A","B"],"default":"A","placement":"target"}]}
```

`placement` is optional; existing `vyre.ui` tabs and unplaced settings remain unchanged. Native controls edit the same digest-bound `api.extensionSettings[extensionId][key]` values as Plugin settings, not a second store. Multiple plugins can contribute controls. Changes are validated by the host and rejected during native calibration or the relevant plugin run; an active calculation uses its captured values. The controls disappear when their plugin is disabled or removed. Existing storage-retention behavior is unchanged. Calibration Settings export/import includes settings from enabled plugins with the `algorithm-replacement` or `engine` capability, including their declared `vyre.settings` fields, whether or not those fields have a placement. Each entry carries the plugin ID and approved digest, legacy settings and extension values. Import applies an entry only to an installed, enabled plugin with the same ID and digest and a matching schema; unavailable, disabled or changed plugins are skipped. Malformed values for a matching plugin are rejected. Import does not install, enable, or reapprove plugins and does not export plugin-private `api.storage`. Plugin code cannot choose CSS selectors, supply HTML, modify the DOM, or access the host renderer. If saved values are invalid under the current schema, use Plugin settings to review and save them before editing the native placements.

### Native placement contract and example

A placement is a property of an individual `vyre.settings` field, not a separate `vyre.ui` view or a plugin-selected DOM location:

| `placement` | Host-owned area | Supported field types |
|---|---|---|
| `calibration` | Calibration Settings | `boolean`, `slider`, `select` |
| `advanced` | Advanced Options | `boolean`, `slider`, `select` |
| `target` | Target Curve Studio | `boolean`, `slider`, `select` |
| `app` | App Settings, General | `boolean`, `slider`, `select` |

The host renders a checkbox, range input or selection menu, respectively, using native UI and the active theme. The plugin ID appears with the label, so contributions from multiple plugins remain attributable. An unplaced setting is still available through Plugin settings. `string` and `number` settings are valid in Plugin settings but cannot declare a native placement. A plugin may not select another location, insert its own markup, style host controls or manipulate the host DOM. `vyre.ui` is the separate declarative plugin-tab mechanism; its input values do not automatically become `vyre.settings` values.

For a standalone declaration in `plugin.json`, the following is a complete manifest shape; it assumes a separately supplied `main.js` function body implementing the declared replacement contract:

```json
{
  "id": "example-processor",
  "name": "Example Processor",
  "version": "1.0.0",
  "author": "Example Author",
  "description": "Illustrative native settings and subwoofer replacement.",
  "apiVersion": 1,
  "capabilities": ["algorithm-replacement"],
  "permissions": [],
  "entry": "main.js",
  "extensions": [
    {"extensionId":"sub", "point":"vyre.subwoofer", "operation":"replace", "contractVersion":1, "permissions":[]},
    {"extensionId":"prefs", "point":"vyre.settings", "operation":"extend", "contractVersion":1, "permissions":[], "settings":[
      {"key":"enabled", "label":"Enable processing", "type":"boolean", "default":false, "placement":"advanced"},
      {"key":"strength", "label":"Strength", "type":"slider", "min":0, "max":10, "step":1, "default":5, "placement":"calibration"},
      {"key":"mode", "label":"Mode", "type":"select", "options":["gentle","normal"], "default":"normal", "placement":"target"},
      {"key":"show_info", "label":"Show information", "type":"boolean", "default":true, "placement":"app"}
    ]}
  ]
}
```

Within the plugin's replacement invocation, read the validated snapshot as `api.extensionSettings.prefs.enabled`, `.strength`, `.mode`, and `.show_info`. These fields do not register callbacks or cause a calibration themselves. Values and the approved plugin digest are captured for the run; the user must run a new calibration after changing them. The host serializes writes, rechecks the owner and digest, and blocks changes while relevant processing is active. Disabled or removed owners lose their native controls; saved values remain. Invalid saved values under a changed schema must be reviewed in Plugin settings before a native control can be edited.

Field keys match `^[a-z][a-z0-9_]{0,31}$` and must be unique within the declaration; reserved keys are rejected. A label is nonempty and at most 80 characters; optional `group` and `description` are at most 80 and 300 characters. A select has 1–20 unique, nonempty string choices of at most 80 characters and its default must be a member. A slider requires finite `min` and `max`; optional `step` must be positive, and default and stored values must lie within the range and on the step grid relative to `min` (or zero when `min` is absent). A boolean default must be a boolean. Unknown fields, placements and mismatched field types are rejected rather than ignored.

Settings are stored in the existing atomic app-settings file. Disabling a plugin or removing its folder retains its data. A changed digest still requires explicit reapproval. Saved values are revalidated against an updated schema: incompatible values remain stored but the UI displays defaults and a warning. Saving a setting intentionally replaces that field; fields absent from the new schema remain preserved within the 32-key limit. No saved data authorizes plugin code. Saving extension settings uses the same bounded frontend queue as extension actions, so an already running UI extension completes first. The host still rejects changes during an active native DSP or engine run; calibration sessions retain their original settings snapshot.

All existing plugin execution paths expose the versioned, plugin-private `api.storage` capability: `version` is 1, `get(key)` returns a detached JSON value or `null` when absent, `set(key,value)`, `delete(key)` and `keys()` operate only on that plugin's namespace. Keys use 1–64 ASCII letters, digits, `.`, `_` or `-`, start with an alphanumeric character, and may not be reserved object keys. Values are finite JSON primitives, arrays or plain objects; nested reserved keys and executable values are rejected. Limits are 128 keys, 65,536 UTF-8 bytes per plugin, 1 MiB total, nesting depth 16 and 8192 nodes per value. There is no filesystem path or additional OS permission.

Storage changes are committed atomically only after a successful, validated plugin result and a fresh digest/enablement check. Concurrent writes against a changed snapshot fail rather than overwrite each other. Data remains after disable or folder removal until the app data is removed; there is currently no per-plugin storage deletion UI. For native calibration, replacement stages of the same owner share a binding with fixed settings and a consistent storage snapshot; a conflicting external storage change aborts rather than silently changing the running calculation.

## Read-only measurement bridge

To request access, add `"measurements:read"` to the **plugin manifest's** `permissions` array; the individual universal extension declaration still uses `"permissions":[]`. The enable dialog discloses the request, and changing the manifest requires digest reapproval. This grants no filesystem, network, Node or operating-system permission. Enabled legacy API 1 and universal-extension sandbox actions receive `api.measurements` with `apiVersion:1`, a `generation` string or `null`, and an asynchronous `query(request)`. Engine API 2 is unchanged.

For example, from an asynchronous plugin function:

```js
const generation = api.measurements.generation;
const info = await api.measurements.query({apiVersion:1, generation, type:'metadata'});
const samples = await api.measurements.query({apiVersion:1, generation, type:'impulse', channel:'FL', position:'0', offset:0, limit:512});
```

Metadata identifies the original input, receiver label, declared subwoofer mode, layout and count when present, and actual channel IDs, roles and position lengths. Routing fields describe the original ADY declaration; they are not a promise that a receiver supports that routing. An SWMIX channel remains one `aggregate-subwoofer` channel; no independent paths are created. Impulse chunks contain finite original samples with their offset and total length. Frequency, phase, complex spectra, calculated results and REW-imported measurements are not yet exposed. No values are invented.

The renderer transfers at most 4096 samples per message to an immutable main-process snapshot (up to 16,777,216 samples, 64 channels, 64 positions per channel and 262,144 samples per position). A transfer expires after 30 seconds without a chunk. A sandbox run is bound to the generation present at its start and may issue at most 128 queries, each returning at most 4096 impulse samples. Loading, removing or changing projects invalidates old generations; a snapshot can temporarily be unavailable while transfer is in progress. The main process verifies the sending sandbox, active plugin capability, explicit permission and current approved digest on each request. Existing sandbox timeout, cancellation and plugin-disable handling apply. UI actions remain unavailable during native calibration.

Metadata also reports `sampleRateHz:48000` with `sampleRateSource:'native-processing-assumption'`: this is the rate used by the existing native processing path, not an independently measured or ADY-declared rate. Plugins perform any frequency or phase analysis themselves.

## Calibration observers

Declare for example `{"extensionId":"watch","point":"vyre.calibration","operation":"observe","contractVersion":1,"permissions":[],"events":["measurement.loaded","measurement.cleared","calibration.started","calibration.completed","calibration.failed","calibration.cancelled"]}`. The manifest retains `algorithm-replacement` as its existing execution capability. A plugin may declare multiple uniquely named observers. Each sandbox invocation has `api.mode:'observe'`, `api.stage:'calibration-observe-v1'` and `input.event`, a versioned object containing `apiVersion:1`, `type`, and, where available, a calculation `session` and measurement `generation`. Return `{apiVersion:1}`; use the existing private `api.storage` for plugin state. A storage change refreshes that owner's existing UI extensions after validation, without exposing native UI objects.

`measurement.loaded` is emitted only after snapshot transfer succeeds; `measurement.cleared` follows invalidation. Calibration events cover actual native full-calibration start, completion, failure and user cancellation. Electrical-only exports and independent Engine API 2 projects are not native calibrations and do not emit these events. Old measurement generations cannot be queried after a project change. Measurement queries still require the explicit `measurements:read` manifest permission; without it a plugin may receive lifecycle events but cannot read measurements.

Events are dispatched in owner/extension order to isolated per-observer queues, with at most 16 pending events per observer, 64 registered observers and four concurrent sandbox executions. Overflow is dropped and reported to the host log; observers have the existing five-second sandbox timeout and memory watchdog. Disable, removal or digest changes cancel active work and remove subscriptions. Observer errors are isolated and logged. To avoid perturbing native replacement sessions, observer storage commits made during calibration are deferred until the native run terminates; conflicting or invalidated commits fail closed. UI refreshes are deferred while native calibration is busy. Events do not wait for observers.


## Host integration and future points

`universal-extensions.cjs` contains the explicit point/operation/version-to-stage and adapter catalog. The dispatcher normalizes declarations and delegates ownership/conflict checking to the existing replacement registry. `fromPlugins` is shared by selection and execution, and the existing sandbox validates the selected adapter's input and output. A future supported replacement point requires an explicit catalog entry, versioned validation adapter and a host integration at the correct native boundary; the registry, sandbox and dispatcher do not need point-specific branches. Legacy stage declarations continue through their existing contracts.

The `vyre.measurements` provider is a separate, single-owner input extension; it is distinct from the read-only measurement bridge. Unknown IDs or operations are rejected rather than passed host objects or guessed capabilities.

## Local REW capability

A plugin may explicitly request the manifest permission `rew:api`. Enabling it requires the existing digest-bound user approval, whose dialog states that the plugin may communicate with local REW and change permitted REW settings. Adding or removing the permission changes the digest and requires fresh approval. The permission is independent of `measurements:read`; it does not grant native VYRE measurements, other plugins' storage or general network access. `rew:write` separately authorizes filter edits and direct measurement-data imports; `rew:files` separately permits asking the user to select individual files for import or export. Both require `rew:api`. Adding either permission changes the digest and requires a new explicit approval; existing `rew:api` approvals do not acquire these rights. The capability is available as `api.rew`, with `api.rew.apiVersion === 1`, in the existing plugin/extension sandboxes and Engine API 2.

Call `await api.rew.request({apiVersion:1,method:'GET',path:'/measurements'})`. The result is `{apiVersion:1,status,data}`, where `data` is parsed JSON or `null` for an empty response. Optional `query` is an object of string values with only `unit`, `smoothing`, `ppo`, `windowed` and `normalised` keys. The host connects directly to the fixed IPv4 literal `127.0.0.1:4735`; plugins cannot supply the host, port, headers, credentials or proxy. Redirects are rejected, not followed. No response is executed or used to construct further requests automatically.

The supported subset is deliberately limited to documented, reviewed REW calls:

- `GET /measurements`, `/measurements/process-result`, `/measurements/process-measurements-result`, `/measurements/frequency-response/units`, `/measurements/frequency-response/smoothing-choices`, `/measurements/impulse-response/units`, `/eq/equalisers`, `/eq/manufacturers`, and `/application/blocking`.
- `GET /measurements/{id}/{frequency-response|impulse-response|group-delay|target-response|target-level|target-settings|equaliser|filters|ir-windows}` and `/measurements/{id}/eq/{frequency-response|group-delay|impulse-response}`; IDs are numeric or canonical UUIDs.
- With `rew:write`, `PUT /measurements/{id}/filters` accepts one validated REW `FilterSetting` and `POST` accepts `{filters:[...]}`. The schema uses `index`, `type`, `enabled`, `isAuto`, `frequency`, `gaindB`, `q`, `t60Target`, `frequency2`, `q2`, `shape`, `slopedBPerOctave`, and `slope2dBPerOctave`; unsupported fields, duplicate indices and invalid numbers are rejected. REW and its selected equaliser remain responsible for supported filter types.
- With `rew:write`, `POST /import/frequency-response-data` accepts the documented `FrequencyResponseData` fields, and `POST /import/impulse-response-data` accepts `ImpulseResponseData`. Arrays must be canonical Base64 of finite, big-endian 32-bit floats; frequency magnitude and optional phase lengths must agree. The corresponding `GET` import status endpoints require `rew:write`.
- `POST /application/blocking` accepts a JSON boolean body. `POST` or `PUT /measurements/{id}/ir-windows` accepts only the documented `leftWindowType`, `rightWindowType`, `leftWindowWidthms`, `rightWindowWidthms`, `refTimems`, `addFDW` and `fdwWidthCycles` fields, with validated scalar types. These writes alter REW state; `/application/blocking` is global to REW. A timeout or connection cancellation cannot undo an operation already accepted by REW.

With `rew:files`, `await api.rew.importFile({apiVersion:1,format:'frequency-response'|'impulse-response'|'mdat'})` prompts the user to choose one `.txt`/`.frd`, `.wav`/`.aif`/`.aiff`, or `.mdat` file respectively. `await api.rew.exportFile({apiVersion:1,format:'mdat',measurementId:'1'})` prompts for a new `.mdat` destination. Neither accepts or returns a filesystem path. The host reads the selected regular non-symlink file, stages it privately, and sends REW only its generated temporary path; export uses only the fixed single-measurement Save operation. The destination is published exclusively without overwriting an existing file, including a race where it appears after selection. Export requires a filesystem that supports hard links. Files are limited to 128 MiB (134,217,728 bytes), checked both before and during bounded 256 KiB streaming copies. Changes to source size or high-resolution modification/change metadata during copying abort the transfer; this is not protection against a privileged process that can circumvent filesystem metadata. Completion must be verifiable from the documented import status or a matching REW operation ID; unverifiable asynchronous results fail closed. A single file operation is active globally, each sandbox may request at most eight, and the user dialog and transfer have a two-minute overall deadline. Individual file requests and completion polling remain limited to 30 seconds, which can cause large or slow REW operations to time out. The normal plugin execution timer is paused only during that bounded, user-visible file operation. Cancellation, plugin disable/removal or digest change invalidate it; REW may still complete work already accepted.

All other arbitrary command bodies, subscriptions, plugin-supplied local filenames, generator or measurement control, and callback/`resultUrl` fields remain rejected. The host alone constructs the reviewed file-specific REW Load/Save commands. This is not a complete REW API client. REW documents callbacks and file operations in its [official API reference](https://www.roomeqwizard.com/help/help/html/api.html); the fixed destination alone would not make arbitrary operations safe.

Ordinary requests are limited to 16 KiB bodies, direct measurement imports to 2 MiB, JSON responses to 8 MiB, ordinary requests to three seconds, two concurrent requests per plugin sandbox, eight globally and 64 ordinary requests per sandbox lifetime. Host-mediated file requests use bounded 30-second network operations and a separate bounded polling budget. Invalid JSON/UTF-8, overlong responses, unreachable services, timeouts and invalid requests fail in a controlled way. Plugin disable/removal, permission or digest changes and sandbox cancellation close outstanding connections. The host cannot guarantee cancellation of REW work already underway. REW has no authentication at this address: another local process could impersonate the service on port 4735. Grant access only to trusted plugins and a trusted local REW instance.

## Plugin installation and activation

Place a valid plugin folder in the Plugins directory. VYRE discovers ordinary plugins and requests the existing security approval; accepting it activates the approved digest without a second Enable click. Updates, including permission changes, require approval of the new digest. Refusal leaves the plugin inactive; Enable lets the user retry. Disable persists across restarts and updates. Existing inactive installations are conservatively treated as disabled when migrating older settings, because those settings did not record why they were inactive.

Engine-only plugins remain manually enabled. A plugin with both ordinary extensions and an Engine API 2 engine may have its ordinary capabilities enabled after approval, but the active engine is never changed automatically. The user must explicitly select a plugin engine. If an unavailable engine is still selected, its plugin must be manually re-enabled, avoiding automatic engine reactivation. Stage ownership conflicts prevent activation and are reported. Removing the plugin folder cancels active use and unregisters its extensions; stored settings and private storage remain. There is no in-app uninstall action.

## Engine selection and default

A fresh installation defaults to the native engine. The existing persisted `engine` setting is the default; earlier user selections remain preserved. Engine selection in the existing settings UI is session-only. **Set as Default Engine** validates the selected engine's availability, enabled status and current digest, then atomically saves its identity in the existing settings store. The stored default is the session selection on the next application start. Neither installing nor approving an engine plugin changes the selection or default.

An unavailable, disabled, removed or changed default remains stored and is shown as unavailable. No plugin is automatically reactivated or reapproved, and no native fallback is performed. A user can explicitly select native or another available engine for this session and explicitly set it as default; changing only the session does not overwrite the saved default. A plugin update requires approval of its new digest before it can execute. Switching or changing the default is blocked during active host operations and while a plugin engine session exists; the UI also requires closing the current project first. Existing plugin errors remain errors rather than switching engines.

Executed engine-selection and lifecycle checks and their limitations are recorded in `SDK-GESAMTABNAHME-TESTBERICHT.md`.

# Native algorithm replacement contracts

Engine API 2 runs an independent whole-engine project. Algorithm replacements instead run one registered native stage inside the existing isolated plugin renderer; the native engine orchestrates calibration and exports the receiver data. The implemented stages are `crossover-v1` and `subwoofer-optimization-v1`. Separate phase, delay, level and EQ stages are not implemented.

The legacy declaration below and universal declarations both remain supported.

## Declaration

A plugin uses the standard manifest and `main.js`, no permissions, and adds:

```json
{
  "capabilities": ["algorithm-replacement"],
  "permissions": [],
  "replacements": [{"replacementId":"my-crossover","stage":"crossover-v1","apiVersion":1}]
}
```

The usual ID, name, version, author, description, API version and entry fields remain required. An enabled stage has exactly one owner; a conflict aborts the operation. The plugin ID and manifest/code digest are bound to the calibration session. Disabling or changing the plugin, cancellation, timeout, invalid result or conflict aborts calibration without falling back to native selection. Plugins execute in the existing isolated renderer without Node, filesystem, or network permission.

## Crossover contract

`main.js` receives `input` and frozen `api`. `api.capability` is `algorithm-replacement`, `api.mode` is `replace`, and `api.stage` is `crossover-v1`. The input has `apiVersion`, `stage`, `context` (session and candidate UUID, receiver ID and routing mode), `stagesHz`, `frequenciesHz`, `targetDb`, and `channels`. Channels contain their ID, speaker/subwoofer role, distance, level, polarity, correction spectrum, speaker safety minimum, and positions with complex `[real, imaginary]` samples on the shared frequency grid. These snapshots do not expose solver objects. No native selected crossovers are supplied.

The plugin returns `{apiVersion:1, choices:[{channelId, frequencyHz}, ...]}` for every speaker. The host requires supported receiver stages, the measured safety minimum, no duplicate/missing/extra speaker, and the same value for supported left/right pairs. It rejects other returned fields. The native selector and its subsequent universal selection are skipped; native fixed-choice evaluation and dependent distance/export processing use the replacement choices.

The stage supports one to four independently measured acoustic paths in Standard routing; supported native Directional layouts (`Left/Right`, `Front/Rear`, `FL/FR/Rear`, `FL/FR/RL/RR`); and supported preserved-receiver SWMIX aggregates with two to four physical outputs. The optional versioned `routing` snapshot contains `mode`, `layout`, `measurementType`, `physicalCount`, measured `subwooferIds`, and native `speakerRoutes`; aggregate input contains one measured SWMIX, not fabricated physical measurements. `receiver` includes the native profile capabilities, and `settings` supplies relevant crossover flags. Each channel includes all available measured positions and, when present, its own target spectrum. Legacy API 1 input/output and stage IDs remain valid; the additions do not change the result shape.

A plugin may own both stages, or separate non-conflicting plugins may own them. For independent multi-sub combinations, the crossover plugin runs once before the subwoofer plugin; the selected frequencies are passed to the latter through its existing speaker crossover fields. This avoids using native seed choices for the plugin subwoofer optimization. Subsequent native processing only evaluates the fixed plugin crossovers; the universal and SWMIX crossover refits do not replace them. SWMIX subwoofer decisions use the existing shared-EQ-only contract; its physical receiver settings remain preserved.

Manual crossover overrides conflict and are rejected; electrical-only workflows, electrical SW4, unsupported aggregate forms, missing individual measurements, ambiguous routing and Directional conversion fail closed. Input remains bounded to 1 MiB and 16 positions per channel; crossover input has 97 frequency bins, at most 20 channels and at most 20 receiver stages. Receiver profiles and route mappings describe the software model, not verified hardware behavior.

## Single-subwoofer optimization contract

Declare `stage:"subwoofer-optimization-v1"` and `apiVersion:1` using the same `replacements` manifest array. Only one active plugin may own a given stage. For the single-sub contract, supported input is one configured and independently measured acoustic SW1, SW2 or SW3 in Standard routing. The composite contract below supports two, three or four independently measured acoustic channels in Standard routing. Supported preserved-receiver SWMIX aggregates use the separate aggregate shape described below. Directional uses the explicit routing contract below; electrical SW4 and electrical-only workflows fail closed. The crossover replacement can be used independently or together with this stage in supported topologies.

The host supplies `context` (session and candidate UUID), `channelId`, `sampleRateHz` (48000), `speedOfSoundMps`, `frequenciesHz` (121 bins, 20–500 Hz), `targetDb`, `micCorrectionDb`, and `channels`. Each measured channel has `id`, `role`, and `positions` with `key` and signed complex `[real, imaginary]` measurement spectra. `limits` gives `maxBoostDb`, `noInversion` and the valid distance window; `settings` gives manual subwoofer PEQ, speaker distances and room-mode cut settings. These are measurement and configuration snapshots, not native subwoofer solutions. Values are bounded before they reach the sandbox.

Return exactly `{apiVersion:1,channelId,distanceM,levelDb,polarity,filters}`. Distance must be within the supplied receiver window, level within ±12 dB in 0.5 dB steps, polarity `1` or `-1` (`-1` is rejected when inversion is disabled), and `filters` contains up to 20 RBJ `{type,freq,gain,q}` filters. Types are `PK`, `LS` and `HS`; valid frequency is 10–20000 Hz, gain −20 to +10 dB and Q 0.1–10. The host computes complex filter response with the native 48-kHz model, checks every native output bin against automatic boost/cut and combined PEQ/headroom limits, and rejects invalid results. Manual PEQ remains separately combined by the native host.

The stage replaces native single-sub EQ, joint search and later delay/phase/trim refinements; the resulting bank is evaluated at fixed choices for diagnostic polarity evidence and passed to the native crossover and ADY processing. ADY stores sampled EQ magnitude, not a biquad bank or a new polarity toggle. Polarity is a recommendation relative to the measured setup; actual AVR phase reconstruction is not verified. Native speaker optimization and crossover processing remain active. Plugin failure, disabling and cancellation abort the calibration; no native subwoofer fallback is used.

## Two to four independently measured subwoofers (same stage, API 1)

For two to four independently measured standard-routing paths `SW1` through `SW4`, the existing stage runs once for the whole group. Instead of `channelId`, input contains a `channelIds` array identifying all participating subs; `channels` includes all subwoofers and measured speakers with complex spectra at every available position. The shared target, per-subwoofer targets, microphone correction, manual PEQ, speaker distances, speaker EQ/trim/crossover and existing receiver/boost limits are supplied as bounded snapshots. The plugin may optimize the paths jointly using its own algorithm; no native joint solution or individually optimized subwoofer choices are provided.

Return `{apiVersion:1,subwoofers:[{channelId,distanceM,levelDb,polarity,filters}, ...]}` with one entry per participating subwoofer. Each channel must occur exactly once; the single-sub numerical limits and headroom checks apply to each result. One-subwoofer input and output remain unchanged. Plugins that do not support the supplied group size fail closed if they do not return the composite result.

The native joint subwoofer search and post-EQ/delay/level/polarity refinements are skipped; native fixed-result evaluation is diagnostic only. The normal native ADY serializer writes the independently measured SW1–SW4 records with their corresponding filter magnitudes, distances and trims. SWMIX aggregate inputs use the separate contract below and are not converted into fabricated independent paths. As with the existing native export, software serialization does not establish that every receiver or Editor applies independent physical EQ or reproduces the plugin's modeled filter phase; external hardware behavior remains unverified. The plugin polarity choices remain relative physical recommendations, not serialized polarity toggles.

## Directional routing (same stage, API 1)

Independently measured Directional paths use the existing composite `channelIds` / `subwoofers` result shape and add `scope:"directional-v1"` and a `routing` snapshot to input. Supported native layouts are `Left/Right` and `Front/Rear` with two independently measured outputs, `FL/FR/Rear` with three, and `FL/FR/RL/RR` with four. The native receiver profile and layout must identify Directional routing; legacy labels that native code resolves to Standard, mixed/aggregate data, missing paths and electrical SW4 fail closed. Directional-to-SWMIX conversion uses the separate shared-EQ scope described below.

`routing` contains `mode`, `layout`, receiver ID and its declared output capacity, `outputs` (channel IDs and channel types), `speakerRoutes` (the native per-speaker output IDs), and `lfeSubwooferIds`. The routes come from native `routedIds` and describe the existing software model; `hardwareRoutingValidated:false` makes their hardware status explicit. `control` describes per-channel EQ, distance and trim, with polarity as a recommendation only. The usual bounded complex position spectra, per-sub targets, microphone curve, speaker EQ/crossover/distance/trim, PEQ and safety limits remain available. Each Directional channel identifies its measurement type and carries `originalReverseFlag` as original ADY metadata (boolean or null); this is not the plugin polarity recommendation. The host checks routing completeness, identity, layout, capacity and response limits. Plugins choose their own algorithm and return one complete result per subwoofer.

Native joint EQ, delay, trim and polarity searches are skipped in this mode; fixed-result evaluations are diagnostic only. Speaker optimization and native crossover scoring remain active. As with aggregate replacement, the later universal crossover refit is skipped in Directional replacement so the group is sent to the plugin once. Existing ADY flags remain unchanged: the polarity output is a recommendation relative to the measured setup, not a serialized extra polarity switch. Actual AVR routing and phase behavior remain unverified.

## Directional to SWMIX (subwoofer stage, API 1)

When Directional-to-SWMIX conversion is selected, the registered subwoofer replacement is invoked once with `scope:"directional-to-swmix-v1"`. The supported layouts are `Left/Right`, `Front/Rear`, `FL/FR/Rear` and `FL/FR/RL/RR`, with two, three or four independently measured subwoofers. The existing stage owner, permissions, sandbox and timeout apply. A simultaneous crossover replacement remains unsupported for this conversion.

Input includes `channelIds`, signed complex spectra for each available speaker and subwoofer position, original reverse flags as metadata, `routing` with physical output IDs, channel types, speaker routes and receiver capabilities, and a 121-point frequency grid. `targetDb` is the shared target; `settings.subTargets` provides individual target spectra. `settings.physicalInputs` contains original distance, trim and reverse flag; `settings.speakers` and `speakerDistances` describe the native speaker solution. The settings also supply `micCorrectionDb`, `receiverMicTransfer` (a complex pair or null per bin), manual PEQ, kick-bass state, target mode and room-mode limits. Existing per-stage input-size and position limits apply. Receiver routing and acoustic phase are not hardware-certified.

### Routing and input snapshot

This is a scope of `subwoofer-optimization-v1`, not a separately registered stage. Declare the existing `vyre.subwoofer` / `replace` extension (or legacy `subwoofer-optimization-v1` replacement) and branch on `input.scope`. `api.capability` is `algorithm-replacement`, `api.mode` is `replace`, and `api.stage` is `subwoofer-optimization-v1`. Handle the other scopes explicitly or return an error; no other plugin or native stage is silently substituted for an unsupported scope.

| Input | Contents and interpretation |
|---|---|
| `apiVersion`, `stage`, `scope`, `context` | Contract 1, stage, scope and session/candidate UUIDs. Treat IDs as opaque. |
| `channelIds` | All participating physical acoustic IDs, consecutively `SW1` through `SW2`, `SW3` or `SW4`; each must appear exactly once in the result. |
| `sampleRateHz`, `speedOfSoundMps` | Processing sample rate 48000 Hz and the receiver profile's speed in m/s. |
| `frequenciesHz`, `targetDb` | 121 increasing frequency bins and the shared target gain in dB on that grid. |
| `channels` | Measured speaker and acoustic-subwoofer records. Each has `id`, `role`, `measurementType`, `originalReverseFlag`, and `positions:[{key,spectrum}]`; a spectrum contains 121 finite `[real,imaginary]` pairs. Position keys are strings, not array indices to invent. |
| `routing` | `mode`, `layout`, `receiver:{id,maxPhysicalSubs}`, `control:{eq:"shared",distance:"per-channel",trim:"per-channel",polarity:"recommendation-only"}`, `outputs`, `speakerRoutes`, `lfeSubwooferIds` and `hardwareRoutingValidated:false`. |
| `limits` | `maxBoostDb`, `noInversion`, `distanceMinM`, `distanceMaxM` and `distanceSpanM`. These are host constraints, not suggested results. |
| `settings` | `speakerDistances`, `physicalInputs`, `manualPeq`, `subTargets`, `kickBass`, `targetMode`, `receiverMicTransfer`, `maxRoomModeCutDb`, `subRoomModeLimitHz`, `speakers`, and `micCorrectionDb`. |

Each `physicalInputs` entry contains `channelId`, source `distanceM`, source `levelDb`, and `originalReverseFlag`; it is input metadata, not an optimized answer. `subTargets` contains `{channelId,targetDb}` arrays on the supplied grid. `speakers` contains `{id,distanceM,levelDb,crossoverHz,eqDb}`; `speakerDistances` contains `{id,distanceM}`. `manualPeq` is keyed by `swmix` and the participating lowercase subwoofer IDs, with validated `{type,freq,gain,q}` filter descriptions. `micCorrectionDb` contains one gain per bin; `receiverMicTransfer` contains either a complex pair or `null` per bin. All arrays and measurements are detached snapshots; they grant no access to the native renderer or objects.

The host accepts only supported Directional layouts and their matching 2–4 independently measured, sequentially identified acoustic channels. Measurements must satisfy the native routing and common-position requirements; an explicitly unavailable shared timing reference is rejected. In individual target mode, participating subwoofer targets must be identical, and active per-output acoustic subwoofer PEQ conflicts with conversion; use the common SWMIX PEQ instead. These native preconditions also apply to the replacement path. The documented `Left/Right` and `Front/Rear` layouts each use two outputs; `FL/FR/Rear` uses three and `FL/FR/RL/RR` four. These are software routing contracts, not a guarantee about a particular receiver's hardware. An aggregate-only input has its own `aggregate-v1` contract; it must not be represented as individual measurements. Electrical-only processing, an unsupported or ambiguous topology, and concurrent crossover replacement fail closed.

Return one decision per physical output and one **shared** EQ filter bank:

```json
{"apiVersion":1,"scope":"directional-to-swmix-v1","subwoofers":[{"channelId":"SW1","distanceM":2.5,"levelDb":-2,"polarity":1},{"channelId":"SW2","distanceM":2.75,"levelDb":-1.5,"polarity":-1}],"filters":[{"type":"PK","freq":80,"gain":-1,"q":1}]}
```

No per-output filter banks or extra fields are accepted. Distances must satisfy the supplied receiver bounds, 0.001 m resolution and common distance-span limit; trims must be within ±12 dB in 0.5 dB steps. Polarity is `1` or `-1` relative to the measured setup; `limits.noInversion` permits only `1`. Filters use the existing bounded PK/LS/HS contract, with at most 20 entries. The host checks combined sampled EQ, manual PEQ, modal-cut and headroom limits. Invalid outputs, cancellation, timeouts, changed ownership or disablement abort the run without native fallback. The host rechecks the digest-bound owner and storage before completion.

The host synthesizes the common response from the original impulses and validated physical choices and serializes the fixed distance, trim and shared EQ values through the existing ADY writer. Native subwoofer and group-parameter optimization are skipped. Physical polarity is a recommendation and is not switched by the ADY. Final Curves and reports use the serialized shared EQ and identify plugin-supplied decisions. AVR reconstruction and actual physical behavior remain unverified.

### Polarity display and physical action

The returned polarity is relative to the unchanged measurement setup: `1` keeps it and `-1` recommends a physical inversion. VYRE does not write a new polarity toggle to the ADY. Original ADY `isReversePolarity` / `originalReverseFlag` values are receiver metadata, not calculated VYRE or plugin recommendations. With Avoid Subwoofer Polarity Inversion active, the host rejects `-1`.

Log, Calibration Dashboard and HTML report use the host's common recommendation processing. Where the final reference and receiver-model evidence support a calculated decision, it can be displayed as an individual action. Where a reliable physical decision cannot be established, the host displays `KEEP CURRENT POLARITY` as a conservative retention recommendation; it must not be interpreted as proof of an individually optimized polarity. A single common SWMIX measurement produces a group recommendation, not invented SW1–SW4 individual decisions. Additional electrical bass shakers are not acoustic polarity inputs. Changes to source timing or failure of the export/model audit can prevent a calculated physical action from being displayed even if a plugin returned a valid sign.

## SWMIX aggregate (same stage, API 1)

The host accepts Standard-routing virtual aggregates identified by the existing `aggregateInput` logic, with exactly one measured aggregate, two to four declared physical outputs and receiver parameters preserved. This includes supported named SW1–SW4 inputs and supported records already labelled SWMIX. The host maps the single measured signal to one logical `SWMIX` entry, `role:"aggregate"`, `measurementType:"aggregate"`; it does not expose empty physical slots as measurements. `physicalCount` identifies the number of declared outputs, not independently observed paths. The input uses `scope:"aggregate-v1"`, `channelId:"SWMIX"`, the same bounded complex frequency/position arrays and speaker, target, microphone, manual PEQ and EQ limits used by the stage. `control` is exactly `{sharedEq:true,individualEq:false,distance:false,trim:false,polarity:false}`.

Return only `{apiVersion:1,scope:"aggregate-v1",channelId:"SWMIX",filters:[...]}`. The single-sub filter limits, combined manual EQ budget and known preserved receiver-trim headroom are validated before continuation. Requests for individual filters, distance, trim, polarity, extra channels or unrecognized fields are rejected. Native shared-EQ and group-parameter optimization are skipped. The existing aggregate exporter writes the validated shared correction according to its existing record mapping and retains physical settings; it does not create individual measurements. Native crossover scoring and aggregate crossover handling still run, but the later universal crossover refit is skipped for this aggregate replacement so that the shared plugin decision is not recomputed or silently selected from multiple plugin runs.

For this `aggregate-v1` scope, SWMIX forms not recognized as preserved-receiver virtual aggregates, externally supplied individual subwoofer measurements, electrical SW4 and Directional aggregates remain unsupported and fail closed. Independently measured Directional input selected for conversion instead uses `directional-to-swmix-v1` as described above. This is intentionally an EQ-only aggregate contract: the existing data and preserved receiver/export path do not supply independently controllable distance, trim or polarity decisions. ADY stores the magnitude curve; actual receiver filter phase and physical response remain unverified.


## Manifest and installation reference

A plugin is a direct child of `userData/plugins/` with `plugin.json` and `main.js`. Use the Plugins folder opened from App Settings rather than hard-coding OS paths. The registry reads regular files, rejects symlinks, duplicates, oversized or malformed manifests and hashes manifest plus code. `entry` must be `main.js`. Each ordinary plugin is approved by the user; after approval it activates automatically unless explicitly disabled. An engine-only plugin is manually enabled. Installation or approval never switches the current or default engine. Updates require approval of the new digest; disablement persists, running work is cancelled and settings/storage are retained.

Complete **manifest example** combining declarations, not a complete runnable plugin:

```json
{
  "id": "example-tools",
  "name": "Example Tools",
  "version": "1.0.0",
  "author": "Example Author",
  "description": "Illustrative declarations only.",
  "apiVersion": 1,
  "capabilities": ["analysis", "algorithm-replacement"],
  "permissions": ["measurements:read", "rew:api"],
  "entry": "main.js",
  "settings": [{"key":"label","label":"Label","type":"string","default":"Example"}],
  "extensions": [
    {"extensionId":"targets","point":"vyre.targetCurve","operation":"extend","contractVersion":1,"permissions":[]},
    {"extensionId":"measurements","point":"vyre.measurements","operation":"extend","contractVersion":1,"permissions":[]},
    {"extensionId":"watch","point":"vyre.calibration","operation":"observe","contractVersion":1,"permissions":[],"events":["measurement.loaded"]}
  ]
}
```

Only these top-level keys are accepted: `id`, `name`, `version`, `author`, `description`, `apiVersion`, `capabilities`, `permissions`, `entry`, `settings`, `formats`, `replacements`, `extensions`. Plugin and extension IDs are lowercase letters, digits and hyphens; versions are numeric `major.minor.patch`. Capabilities: `analysis`, `ui`, `import`, `export`, `engine`, `algorithm-replacement`. Permissions: `measurements:read`, `rew:api`, `rew:write`, `rew:files`; the last two require `rew:api`. Permissions belong to the plugin manifest; each universal extension declaration must currently use `permissions:[]`. Up to 16 declarations per extension/replacement array and 32 manifest settings. Unknown fields, capabilities, permissions, operations and experimental `native-dsp` are rejected.

`main.js` is a **function body**, invoked with `input` and frozen `api`, not a Node module. An asynchronous result may be returned. A safe nonfunctional skeleton is:

```js
throw new Error('Implement the declared contract before enabling this plugin');
```

Do not rely on `require`, `process`, DOM access to the host, direct network or unrestricted files. The runtime checks identity, enablement and digest at operation boundaries. API 1 short-lived calls have a five-second execution timeout, cancellation and memory monitoring; operations and their output are validated. These limits and sandboxing are defense in depth, not a guarantee against runtime vulnerabilities. No automatic native fallback is used for failed native replacements.

Optional `formats` is an object with `import` and/or `export` arrays of at most eight unique `{extension,encoding}` entries per capability, e.g. `{"import":[{"extension":"txt","encoding":"utf8"}]}` with `import` in capabilities. Extension is 1–8 lowercase letters/digits; encoding is `utf8` or `base64`. Host dialogs mediate the files; plugins do not receive arbitrary paths.

## Extension declarations and contract examples

The following declarations may replace the `extensions` array in the complete manifest above; they are data examples, not a solver:

```json
[
  {"extensionId":"cross","point":"vyre.crossover","operation":"replace","contractVersion":1,"permissions":[]},
  {"extensionId":"sub","point":"vyre.subwoofer","operation":"replace","contractVersion":1,"permissions":[]},
  {"extensionId":"range","point":"vyre.crossover","operation":"constrain","contractVersion":1,"permissions":[]},
  {"extensionId":"speaker","point":"vyre.speakerEq","operation":"transform","contractVersion":1,"permissions":[]},
  {"extensionId":"targets","point":"vyre.targetCurve","operation":"extend","contractVersion":1,"permissions":[]},
  {"extensionId":"measurements","point":"vyre.measurements","operation":"extend","contractVersion":1,"permissions":[]},
  {"extensionId":"panel","point":"vyre.ui","operation":"extend","contractVersion":1,"permissions":[]},
  {"extensionId":"prefs","point":"vyre.settings","operation":"extend","contractVersion":1,"permissions":[],"settings":[{"key":"enabled","label":"Enabled","type":"boolean","default":true}]},
  {"extensionId":"watch","point":"vyre.calibration","operation":"observe","contractVersion":1,"permissions":[],"events":["calibration.started","calibration.completed"]}
]
```

Do not enable `cross` and `range` simultaneously: crossover replacement conflicts with constraints. Each native replace stage and the measurement provider have at most one active owner. Multiple constraints and transforms are composed and checked as described above.

Illustrative **return object shapes**; real output must be derived from the supplied invocation and pass all conditions above. These are separate function-body fragments, not a complete plugin:

```js
// vyre.targetCurve / extend: frequencies are Hz and gains dB.
return {apiVersion:1, curves:[{id:'example', name:'Example', points:[[20,0],[20000,0]]}]};
```

```js
// vyre.ui / extend: declarative host-rendered tab, not HTML.
return {apiVersion:1, tabs:[{id:'info', title:'Info', view:{apiVersion:1, components:[
  {type:'paragraph', text:'Illustrative text'},
  {type:'input', id:'note', text:'Note', value:''},
  {type:'button', text:'Apply', action:'apply'}
]}}]};
```

For a UI button invocation, `input.tabId`, `input.action` and `input.values` identify the action and validated inputs; return `{apiVersion:1,view:{apiVersion:1,components:[...]}}`. The view's input IDs must be unique. Declarative settings do not execute a plugin: access their snapshot as `api.extensionSettings.prefs.enabled` in a separately invoked stage. For an observer return `{apiVersion:1}`. A constraints result may be `{apiVersion:1,minHz:80,maxHz:120}` only when the intersection contains supported receiver stages and safety rules permit it.

Crossover output is `{apiVersion:1,choices:[{channelId:'FL',frequencyHz:80},{channelId:'FR',frequencyHz:80}]}` only for an input with exactly these measured speakers and when 80 is supported and safe for both. Subwoofer individual output is `{apiVersion:1,channelId:'SW1',distanceM:3,levelDb:0,polarity:1,filters:[]}` only if the input's distance and safety limits permit it. Multi-sub returns `subwoofers` with one complete object per supplied ID. Aggregate returns `{apiVersion:1,scope:'aggregate-v1',channelId:'SWMIX',filters:[]}`. Real filters must comply with the bounds above. Speaker transforms must return **all** supplied speaker IDs and a complete finite `correctionDb` array on the supplied grid; the per-bin envelope applies. None of these examples determines a valid calibration result for a user's measurements.

For REW, for example, with the corresponding manifest permission and a running trusted local REW instance:

```js
const response = await api.rew.request({apiVersion:1, method:'GET', path:'/measurements'});
return response;
```

The bridge response is not itself a valid result for every extension contract; adapt it to the calling capability. `api.rew.importFile({apiVersion:1,format:'mdat'})` and `api.rew.exportFile({apiVersion:1,format:'mdat',measurementId:'1'})` additionally need `rew:files`. A plugin cannot bypass the user's file picker, provide paths, or obtain general network access.

## Independent Engine API 2

A full engine is independent of native stage replacement and owns its analysis, calibration and ADY production. Minimal declaration (not an implementation):

```json
{
  "id":"example-engine", "name":"Example Engine", "version":"1.0.0",
  "author":"Example Author", "description":"Engine contract illustration.",
  "apiVersion":2, "capabilities":["engine"], "permissions":[], "entry":"main.js"
}
```

Every invocation receives `{apiVersion:2,operation,state,files,...}`; `files` entries contain `{name,size,bytes}` with `bytes` as `Uint8Array` and lazily computed `base64`. The first project file is an ADY; additional user-selected files are transferred once in 1-MiB chunks. Maximum 32 files, 128 MiB per file and 256 MiB per session; renderer memory watchdog is 2 GiB where process metrics are available. State persists in the isolated renderer for the project, including typed arrays; it is lost after a crash or cancellation.

The lifecycle is `initialize`, `getCapabilities`, `getSettings`, `openProject`, `updateSettings`, `calibrate`, `getResults`, `exportProject`, `dispose`. Each returns `{apiVersion:2,state,...}`; `getSettings` and `getResults` may return `message` and a validated `view`. Engine views use `heading`, `paragraph`, `metric`, `button`, `chart`; a button action invokes its operation again with `input.action`. `exportProject` returns `file:{name,bytes}` with a valid `.ady` filename and nonempty output up to 128 MiB; `base64` is also accepted. The host saves the bytes unchanged and does not validate receiver compatibility or the plugin's acoustic calculations.

Engine invocations expose `api.settings`, `api.extensionSettings`, `api.storage`, `api.rew`, and progress/log functions. Measurement Bridge is not part of Engine API 2; the engine has its own input files. For `calibrate`, `api.progress(fraction,message)` accepts 0–1 or `null`; `api.status`, `api.log(message,level)`, `api.warn`, `api.error` report to the shared dashboard. Messages are bounded to 160 characters, 20 per second and 1000 per run, and late messages are discarded. Engine runs have no fixed timeout; cancel destroys the instance and requires reimport. `dispose` is bounded to five seconds. Plugin failure never silently invokes native processing.

Session engine choice is separate from **Set as Default Engine**. A new installation defaults to native; an unavailable or changed saved default stays unavailable until explicit user action. Switching while a project/session is open or a host operation is active is blocked. The native engine is never implicitly swapped in.

## Themes and plugin UI

Plugin controls are rendered by the host and follow the active theme. Plugins cannot inject theme CSS, HTML, scripts or modify the host DOM. Themes are separate packages, not plugin capabilities: their layouts and CSS are validated, and `layout.js` runs in an isolated renderer without access to the host DOM, filesystem, network or Electron bridge. Protected branding, version, navigation and functional controls remain subject to host validation. See [Theme SDK](VYRE_THEME_SDK.md) for the supported layout and styling contracts and their limits.

## Redistribution and third-party notices

The desktop package declares no production npm dependencies; Electron, electron-builder and Playwright are development dependencies. The inspected Windows distribution nevertheless ships the Electron runtime and its bundled Chromium, Node.js, V8 and other components. Preserve `LICENSE.electron.txt` (Electron MIT copyright and permission notice) and `LICENSES.chromium.html` (the bundled components' individual notices and terms) with the installed distribution, and review any component-specific obligations in that notice file. The inspected Windows unpacked output contains both files. This does not imply that the VYRE application itself must be open sourced.

The final Windows NSIS installer redistributes the NSIS stub and its standard nsDialogs and System plugins. See [Third-Party Notices](THIRD_PARTY_NOTICES.md), the accompanying license texts and the FFmpeg source and build-reference package and its remaining verification limits. `electron-builder` and `playwright-core` are packaging/development dependencies, not application runtime dependencies in `package.json`. According to the project owner, the coastal theme backgrounds were generated specifically for VYRE with ChatGPT/OpenAI; see the notices for the scope and limits of the visual review. Community-theme assets and fonts are supplied independently by their authors and must be appropriately licensed.

### Additional serialized shape and size reference

- Measurement `metadata` result is `{apiVersion:1,generation,source:'original',sampleRateHz:48000,sampleRateSource:'native-processing-assumption',receiver,subwooferMode,subwooferLayout,subwooferNum,channels:[{id,role,positions:[{id,samples}]}]}`. An `impulse` query returns `{apiVersion:1,generation,source:'original',type:'impulse',channel,position,offset,total,samples}`; use the actual response field names and returned generation. Position IDs are strings, not filesystem paths.
- Target points are finite `[frequencyHz,gainDb]`, `0 < frequencyHz <= 100000`, `abs(gainDb) <= 1000`; no duplicate frequency and at least two points. Curve `name` is nonempty and at most 120 characters; optional `description` is at most 500.
- UI text is at most 1000 characters, metric value at most 300; chart has 2–512 finite points with absolute coordinates at most 1e9. Input string value is at most 500 characters; select has 1–100 unique options, each at most 200 characters and includes its current value. Tables have 1–20 headers, at most 100 rows, equal row width and strings at most 300 characters. Sliders require finite min/max/value/step, min < max and value on the declared step. Action input carries at most 30 values and at most 8192 serialized characters.
- Crossover replacement input is at most 1 MiB, 97 spectral bins, at most 20 channels and 16 positions per channel; choices cover every supplied speaker exactly once. Subwoofer replacement uses its separate 121-bin low-frequency input and topology-specific outputs described above. Neither accepts extra result fields.
- Manifest JSON and entry source are each at most 64 MiB (67,108,864 bytes). Generic short-lived sandbox results are at most 32,768 serialized characters; speaker-EQ transforms have a 2-MiB serialized limit, and measurement-provider results have a limit of 25,165,824 serialized JavaScript string units. Engine API 2 non-file JSON results are bounded to 2 MiB.

### Measurement providers

`vyre.measurements` is a single-owner `extend` point (contract 1) under the existing `algorithm-replacement` capability. It supplies bounded, host-validated impulse responses to full native calibration while the loaded ADY retains receiver metadata. The plugin receives `api.stage === "measurement-provider-v1"`, `api.mode === "extend"`, and an `input` with `apiVersion`, `stage`, `receiver`, `routing` and channel `{id,index}` descriptors. It must return every requested channel with matching IDs, shared position IDs and valid waveform and timing data. The native engine continues to perform the calibration and export.

For the complete manifest and response contract, sample-rate conversion, shared reference semantics, limits and fail-closed behavior, see [Measurement provider](UNIVERSAL-EXTENSION-API.md#measurement-provider-vyremeasurements-contract-1). Supplying data requires no `measurements:read` permission and does not grant network access; reading original measurements and using REW require their own permissions. This point does not replace an Engine API 2 engine.
