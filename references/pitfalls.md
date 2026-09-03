# Pitfalls

Gotchas observed in practice while using the Homey CLI. Read this when troubleshooting unexpected behavior — and skim it before any High-tier operation.

## Contents

- [Active-Homey discipline](#active-homey-discipline)
- [`broken: false` lies about dead refs](#broken-false-lies-about-dead-refs)
- [`logic:lt`/`gt` conditions compare a cached value, not a live poll](#logicltgt-conditions-compare-a-cached-value-not-a-live-poll)
- [Interactive prompts hang in agent contexts](#interactive-prompts-hang-in-agent-contexts)
- [`--jq` is CLI-side, not local `jq`](#--jq-is-cli-side-not-local-jq)
- [App `name` can be string or i18n object](#app-name-can-be-string-or-i18n-object)
- [`set-capability-value` primitives need `--request-json`](#set-capability-value-primitives-need---request-json)
- [Active-Homey cache is on-disk](#active-homey-cache-is-on-disk)
- [Token-mode for direct IP access](#token-mode-for-direct-ip-access)
- [Cloud API rate limits are shared account-wide, not per-process](#cloud-api-rate-limits-are-shared-account-wide-not-per-process)
- [Update silently overwrites — always back up](#update-silently-overwrites--always-back-up)
- [Self-resetting relays — pulse with `:on`, not `:on` + `:off`](#self-resetting-relays--pulse-with-on-not-on--off)
- [Sun-offset triggers fire **once**, not during a window](#sun-offset-triggers-fire-once-not-during-a-window)
- [`delay` cards can't be cancelled mid-flight](#delay-cards-cant-be-cancelled-mid-flight)
- [`any`/`all` `input[]` need the `::outputSuccess` suffix](#anyall-input-need-the-outputsuccess-suffix)
- ["Enkel" in the UI is `"type": "any"`](#enkel-in-the-ui-is-type-any)
- [Action arguments shouldn't depend on the triggering device](#action-arguments-shouldnt-depend-on-the-triggering-device)
- [Flow Exchanger strings aren't the same schema as `create-advanced-flow`](#flow-exchanger-strings-arent-the-same-schema-as-create-advanced-flow)
- ["Before sunset" / "Before sunrise" mean window, not earlier-than](#before-sunset--before-sunrise-mean-window-not-earlier-than)
- [`cron:after_sunrise`/`after_sunset` are a different, instant-based card family](#cronafter_sunriseafter_sunset-are-a-different-instant-based-card-family)
- [`AND`-push confirm cards are fragile on Android](#and-push-confirm-cards-are-fragile-on-android)
- [Voice assistants only see standard, favorited flows](#voice-assistants-only-see-standard-favorited-flows)
- [Spontaneous flow firing is almost always device-side automation](#spontaneous-flow-firing-is-almost-always-device-side-automation)
- [Risk tiers](#risk-tiers)

## Active-Homey discipline

**The #1 footgun for users with more than one Homey on their account** — rentals, parents' houses, lab units. `homey select` writes the active Homey id to a cache file that persists across terminals and sessions. Any subsequent `homey api …` runs against whatever was last selected.

Failure mode: a script intended for Homey A runs against Homey B because the cache was last set to B in another terminal hours ago. For read ops this is annoying. For write ops (update-flow, delete-device, uninstall-app) this is data loss.

**Rule:** run `homey select current` as the first command of any session and immediately before every Medium- or High-tier operation. Never trust prior selection across the gap between two operations.

To set the active Homey non-interactively:

```bash
homey select --id <HOMEY_ID>     # bare `homey select` is interactive and hangs
homey select current             # verify
```

Anyone sharing the skill: when handing it to a teammate, also share which Homey id is the intended target — the cache is per-machine, not per-account.

## `broken: false` lies about dead refs

A flow can reference a `homey:device:<id>` or `homey:zone:<id>` that no longer exists and still report `broken: false`. Observed first-hand on flows whose trigger zone was deleted and luminance sensor was removed — Homey kept the flow marked healthy. The main automation path silently never fired.

**Implication:** the `broken` flag is a necessary check, not a sufficient one. After every push (and during audits) re-resolve every device/zone reference in the flow's `cards` against live `get-devices` / `get-zones`. See `flow-json-schema.md` for the case study and the jq one-liner.

## `logic:lt`/`gt` conditions compare a cached value, not a live poll

The `homey:manager:logic:lt` / `logic:gt` condition cards compare a `droptoken` (e.g. `homey:device:<id>|measure_luminance`) against whatever value is **currently cached** on that device's capability — not a live read taken when the flow runs. Sleepy/battery Zigbee end devices (motion sensors, contact sensors) only push capability updates when they have something to report, so if the device goes quiet or drops off the mesh, the cached value can sit stale for days or weeks while the flow keeps evaluating against it — silently, with `broken: false` still showing.

Observed: a kitchen motion/light sensor (IKEA VALLHORN) stopped reporting for ~19 days. A flow gating "turn lights on" on `measure_luminance < 10` kept comparing against the last cached value (`11`, not `< 10`) the whole time — the automation looked healthy but never fired.

**Debug check:** for every `droptoken` referenced by a misbehaving flow, run

```bash
homey api devices get-device --id <id> --json --jq '.capabilitiesObj.<capability>.lastUpdated'
```

and compare against current time. A stale timestamp on a capability that should update frequently (motion in a high-traffic room, luminance on a daylight sensor) points at a dead/offline device, not a flow-logic bug.

**Resilience pattern:** don't gate a critical automation on one sensor's cached value alone — see `community-patterns.md#pattern-sensor-independent-is-it-dark-fallback`.

## Interactive prompts hang in agent contexts

Several CLI subcommands fall back to interactive prompts when required flags are missing. Agent contexts have no stdin, so the process hangs until timeout.

Known interactive entrypoints:

- `homey select` (without `--id` or `--name`)
- `homey login` (the OAuth flow itself launches a browser; the retry path can prompt)
- `homey app run` and `homey app publish` (app-developer commands)

Always pass explicit flags. `homey select --id <id>` is the workaround for `homey select`.

## `--jq` is CLI-side, not local `jq`

The `--jq` flag invokes the Homey CLI's bundled jq, not the user's local `jq`. Behavior is *almost* identical but errors surface with the prefix `"jq failed:"` and some less-common operators may differ in version. If a `--jq` expression fails mysteriously, drop `--jq`, output `--json`, and pipe to local `jq` to compare.

A common failure: assuming the output supports `.value.name.en` when the field is actually a plain string in some apps and an i18n object in others — see next pitfall.

## App `name` can be string or i18n object

`homey api apps get-apps` returns each app's `name` as either:

- A plain string: `"Sonos (LocalAPI)"`
- An i18n object: `{"en": "Sonos", "nl": "Sonos"}`

The distinction depends on how the app's manifest is structured. Safe pattern:

```bash
homey api apps get-apps --json \
  --jq 'to_entries | map({
    id: .key,
    name: (if .value.name | type == "object" then (.value.name.en // .value.name.nl) else .value.name end)
  })'
```

## `set-capability-value` primitives need `--request-json`

When the body is a primitive (`true`, `false`, `0.5`), pass `--request-json` so it's sent as a JSON value rather than a literal string.

```bash
# Correct
homey api devices set-capability-value --device-id <id> --capability-id onoff \
  --body 'true' --request-json

# Wrong — sends the string "true", which Homey rejects as a boolean capability
homey api devices set-capability-value --device-id <id> --capability-id onoff \
  --body 'true'
```

For object bodies (`{"key":"value"}`) the default is already JSON, so `--request-json` is redundant but harmless.

## Active-Homey cache is on-disk

The active Homey id lives in a per-machine cache directory (managed by the `homey` binary). Implications:

- Selection persists across terminals and shells on the same machine.
- Selection does **not** transfer to another machine — fresh machine needs `homey login` + `homey select --id` again.
- CI / containers need to either ship the cache or `homey select --id` in their bootstrap step.
- Multi-user machines share the cache per user account (since it's user-scoped).

## Token-mode for direct IP access

`homey api … --token <bearer> --address http://<homey-ip>` bypasses cloud resolution and hits the Homey directly. Useful when:

- The cloud is slow or down.
- Operating from inside the home network without internet.
- Using a guest token (non-owner access).

The bearer token comes from `homey api sessions create-session` or from the Homey developer tools.

## Cloud API rate limits are shared account-wide, not per-process

`homey api …` calls go through Homey's cloud, and its rate limit is scoped to the **account/connection**, not to the calling process or terminal. Running multiple concurrent CLI sessions against the same Homey (e.g. several agent sessions in parallel) can trip a shared `{"error": "Too many requests."}` (429) that keeps returning for **all** of them, even ones that stop calling — because each other session's traffic keeps the shared budget consumed.

Observed: three concurrent Claude Code sessions against the same Homey each hit persistent 429s; the throttle didn't clear after ~5 minutes of total silence across all three, and took roughly 20+ minutes of full silence before any of them succeeded again — consistent with a longer-duration (possibly hourly-scale) account quota, not a short per-minute limiter.

**If you hit persistent 429s:**

1. Check for other active sessions before assuming it's a bug — a peer running the same skill against the same Homey is the most common cause.
2. Don't backoff-and-retry solo if others might be doing the same — each retry resets the shared window. Coordinate a shared pause instead.
3. Expect the cooldown to take tens of minutes, not seconds.
4. After a suspected clear, send one lightweight probe (`homey api zones get-zones --json --jq 'keys | length'`) before resuming normal call volume.

## Update silently overwrites — always back up

`update-flow` and `update-advanced-flow` replace the entire flow body — they are PUT, not PATCH. Sending an object missing a field deletes that field. A typo in the `name` field renames the flow.

**Always back up first** for any write operation that takes a `--body`:

```bash
BACKUP="/tmp/homey-backup-$(date +%s).json"
homey api flow get-advanced-flow --id <id> --json > "$BACKUP"
# … then update …
```

For destructive ops (`delete-*`), the backup is the only way back if the user reconsiders.

## Self-resetting relays — pulse with `:on`, not `:on` + `:off`

Some devices (notably **Shelly** sockets and most garage-door / impulse relays) auto-reset their relay after a configured hold time. Their `onoff` capability mirrors the relay state, **not** the downstream load's state. Two consequences when scripting them in flows:

1. **A single `:on` action is enough for a pulse.** Don't pair it with a delayed `:off` — the relay will already have dropped on its own, and the explicit `:off` either no-ops or fights the device. Multiple `:on` actions in a row (with `delay` cards between) reproduce the physical button-press pattern cleanly — e.g. a garage "package mode": `:on → delay 10s → :on → delay 2min → :on` (open half / hold / close).
2. **`onoff` gives no feedback about the load.** A garage motor that fails to move still reports `onoff: false` once the relay drops. Don't condition follow-up logic on the device's capability state; if you need real position feedback, use a separate door / contact sensor.

Worked example: `flows/garagedeur-pakketmodus.md` (in this project's repo) chains three `:on` actions on a Shelly socket without any `:off` and works reliably.

## Sun-offset triggers fire **once**, not during a window

The card "When the sun sets in N minutes" (`is_sun_event_in` / similar) is a **trigger**: it fires once at `sunset - N`. People consistently mistake it for "during the hour before sunset" and put it as the entry of an AND gate with a motion sensor — which then dead-locks unless motion happens within milliseconds of that single instant.

**Right pattern:** trigger on the activity (motion, button, presence), put the time-window as a **condition** card (`is_sun_event_between` with offsets). Worked example: `assets/flow-templates/examples/motion-sun-window-light.json`. Recurring in the community — see `references/community-patterns.md`.

## `delay` cards can't be cancelled mid-flight

Once a `delay` card emits, the downstream action will fire even if the conditions that placed the flow there reverse. Concretely: motion triggers `delay 60s → light off`. New motion arrives at second 30. The light still goes off at second 60 — there is no native "cancel pending delay" card.

**Right pattern:** named, cancellable timers via the Chronograph app (`start_timer` re-arms a same-named timer, `stop_timer` cancels it). Worked example: `assets/flow-templates/examples/cancellable-off-timer.json`.

If Chronograph isn't installed, fake it with a Logic boolean and an `is_unchanged_for` condition — uglier but works without dependencies.

## `any`/`all` `input[]` need the `::outputSuccess` suffix

The synchronisation cards reference upstream cards by **port**, not by card id:

```json
"input": [
  "<source-uuid>::outputSuccess",
  "<source-uuid>::outputTrue"
]
```

A bare UUID without `::outputName` is accepted by `create-advanced-flow` but the gate never resolves at runtime — the flow appears healthy in the editor and silently does nothing. Output names are `outputSuccess`, `outputTrue`, `outputFalse`, `outputError`.

## "Enkel" in the UI is `"type": "any"`

The Dutch UI label "Enkel" (English "Only") corresponds to JSON `"type": "any"`, not `"only"`. The most common typo when hand-writing advanced-flow JSON. The validator accepts `"only"` (silently treated as unknown type), the runtime ignores the card, and the gate downstream of it never fires.

Same trap inverted: `"type": "all"` is **not** the UI's "AND" — `all` is a join node that waits for every input branch. The And-card (condition) is `"type": "condition"`.

## Action arguments shouldn't depend on the triggering device

A surprisingly common community footgun: an alarm flow's action card pulls a token (volume, target, message text) from the **device that caused the trigger**. When that device fails (dead battery, lost Zigbee link), the trigger never fires *and* the action arg can't resolve when it does — the action either no-ops or errors silently.

**Right pattern:** pull action arguments from neutral sources — Logic variables, a dedicated config flow, a separator/delay node that re-resolves. The smoke-detector escalation thread is the canonical example: route the alarm into a `Simple Log` event with severity, dispatch in a separate flow that pulls volume / target from its own variables.

See `references/community-patterns.md#pattern-smoke--alarm-escalation`.

## Flow Exchanger strings aren't the same schema as `create-advanced-flow`

Strings shared in the community starting with `H4sIAAAA…` are gzip+base64-encoded JSON, but the embedded schema is the **compacted** Flow Exchanger form (`{u, a, c, i, n, e, …}`) — not the full advanced-flow JSON. Decoding gives you something like:

```json
{ "u": { "<uuid>": { "n": "Author Name" } },
  "a": [ { "i": "<uuid>", "n": "Flow name", "e": true,
           "c": { "<uuid>": { "x": 0, "y": 100, "o": "homey:manager:mobile",
                              "i": "push_text", "t": "action", "a": {} } } } ] }
```

`homey api flow create-advanced-flow --body` rejects this directly — it needs the full schema (see `flow-json-schema.md`). Translation rules: `c → cards`, `t → type`, `i → id` (or card id depending on context), `o → ownerUri` (often merged into `id`), `a → args`, `x/y → x/y`.

When the user pastes such a string, decode and explicitly translate before pushing. Don't blindly forward.

## "Before sunset" / "Before sunrise" mean window, not earlier-than

The Logic time-condition cards `is_before_sunset` and `is_before_sunrise` describe a **window**, not an instant comparison:

- `is_before_sunrise` is true between sunset and sunrise (i.e. "during the night").
- `is_before_sunset` is true between sunrise and sunset (i.e. "during daylight").

A flow that reads "AND it's before sunrise AND it's before sunset" via OR is true 24/7 and fires all night. The community recurringly reads these as "the current time is earlier than today's sunrise" — it isn't.

**Right pattern:** single `is_sun_event_between` card with explicit offsets, or two `is_after`/`is_before` cards combined with a single AND. See [how-homey-implements-sunset-and-sunrise/151358](https://community.homey.app/t/how-homey-implements-sunset-and-sunrise/151358).

## `cron:after_sunrise`/`after_sunset` are a different, instant-based card family

Don't confuse these with the Logic app's `is_before_sunset`/`is_before_sunrise` window cards above. `homey:manager:cron:after_sunrise` and `homey:manager:cron:after_sunset` are separate condition cards (`ownerUri: homey:manager:cron`) with no `args` — the toggle between "after" and "before" in the card's title (`It's !{{after|before}} sunrise`) is controlled by the flow JSON's `inverted` boolean, not a dropdown arg:

- `after_sunrise`, `inverted: true` → true before today's sunrise.
- `after_sunset`, `inverted: false` → true after today's sunset.

These read as simple instant comparisons (not the sunset-to-sunrise window the Logic cards above describe), and are astronomically calculated server-side, so they never go stale like a physical sensor. See `community-patterns.md#pattern-sensor-independent-is-it-dark-fallback` for a resilience pattern that OR-combines them with a luminance condition.

## `AND`-push confirm cards are fragile on Android

The "Push notification (CONFIRM)" condition card asks the user a Yes/No via push and continues based on the answer. Recurring report (Samsung Fold 7, Pro 2023): after 2 confirmations the Yes/No buttons stop appearing — the notification arrives but is silent on long-press, app reinstall fixes it for another 2 messages.

**Implication:** do **not** put security-critical decisions ("turn off the alarm? yes/no") behind confirm-push as the only path. Either:

- Confirm via a physical device action (button press, NFC tap, Hue Tap Dial) that posts a separate trigger.
- Confirm via Homey's voice prompt + Sonos / Google announcement.
- Send a regular push with a deeplink to a dashboard switch and require the user to flip the switch.

The confirm card is fine for low-stakes prompts ("turn off all lights? yes/no"). Don't put it on the critical path.

Source: [push-notifications-confirm-issue/133080](https://community.homey.app/t/push-notifications-confirm-issue/133080).

## Voice assistants only see standard, favorited flows

Alexa, Google Assistant, and Siri integrations expose **standard flows marked as favorite**, not advanced flows. An advanced flow named "Goodnight" does not appear in Alexa's "Routines" scene picker. Disabling a favorited standard flow doesn't always purge it from Alexa's cache — the user has to re-scan scenes.

Pattern: wrap advanced flows in a thin standard flow `[Virtual Button] → Start a flow → <advanced flow>`, mark as favorite. Virtual Device class matters — `light` and `switch` are reliably recognised; `button` is hit-or-miss.

See `references/community-patterns.md#pattern-voice-assistant--advanced-flow-bridge`.

## Spontaneous flow firing is almost always device-side automation

When the user reports "the light came on but no flow fired" or "the heater turned off but the schedule says it shouldn't have", the cause is almost never a phantom Homey flow. Check first:

1. **Hue Bridge** has its own motion + time routines configured via the Hue app — independent of Homey.
2. **Philips / IKEA / Aqara hubs** carry their own automations even after pairing to Homey.
3. **Z-Wave association groups** can wire a motion sensor directly to a light at the device level, bypassing the controller.
4. **Zigbee Touchlink** sometimes leaves direct bindings between devices.
5. **The device's own schedule** (most smart thermostats, washing machines, etc.).
6. Another **flow with `trigger-flow` action** elsewhere — use the Find-Any-Items audit script (see `community-patterns.md#audit-before-refactor-who-references-this-device`).

Disable Homey's flow temporarily; if the behaviour persists, the source is external. This saves hours of "is my flow broken?" debugging.

Source: [licht-gaat-zelf-weer-aan-zonder-flow-beweging/73832](https://community.homey.app/t/licht-gaat-zelf-weer-aan-zonder-flow-beweging/73832).

## Risk tiers

The full Safe / Medium / High table with required steps lives in `SKILL.md` under the "Risk tiers" heading. Treat it as the single source of truth.

When in doubt, treat the operation as one tier higher than feels intuitive. Smart-home state is hard to roll back — every device that depends on a flow inherits that flow's correctness.
