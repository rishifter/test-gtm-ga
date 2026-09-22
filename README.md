# test-gtm-ga

Proves that one GTM container can push the same event to **two GA4 properties**.

Live: https://rishifter.github.io/test-gtm-ga/

## Try it

Open the live URL, or run it locally:

```bash
python3 -m http.server 8000
```

Then <http://localhost:8000>. GTM will not load over `file://`.

Open DevTools → Network and filter on `collect`. Click a button.

**Two hits per click — same `en=`, different `tid=`** means the fan-out works.

## GTM container setup

Container: **`GTM-NW92C2WC`** (already wired into `index.html`). Build four tags in it:

| Tag | Type | Setting | Trigger |
| --- | --- | --- | --- |
| Tag 1 | Google Tag | `G-AAAAAAA` | Initialization - All Pages |
| Tag 2 | Google Tag | `G-BBBBBBB` | Initialization - All Pages |
| GA4 Event 1 | GA4 Event | Measurement ID `G-AAAAAAA`, event name `{{Event}}` | Custom Event, regex `^test_` |
| GA4 Event 2 | GA4 Event | Measurement ID `G-BBBBBBB`, event name `{{Event}}` | same trigger |

Two Google Tags = two configured destinations. GA4 Event tags target a single measurement ID each, so one per destination is the supported path.

You can point both Google Tags at **fake** measurement IDs and still see the two hits — the routing is observable without owning either property.

## Verifying

1. **Network tab** — the actual test. Two `collect` hits per click.
2. **GTM Preview** (Tag Assistant → `http://localhost:8000` or the Pages URL) — shows which tags fired and why.
3. **GA4 DebugView** — only if you want to confirm data lands. Tag Assistant sets `debug_mode` for you.

Layer 1 is the test; 2 and 3 are confirmation.

---

# Developer notes

Full setup walkthrough.

## 1. GA4 properties

You need two measurement IDs. **Or don't** — two invented IDs (`G-TEST111111`, `G-TEST222222`) prove the routing perfectly well, because GA4's collect endpoint accepts any `tid`. Real properties only buy you DebugView.

To create them for real, at <https://analytics.google.com> → **Admin** (gear, bottom-left):

1. **Account** column — pick one, or **Create → Account**.
2. **Property** column — **Create → Property**.
3. Name it `GTM Test – A` (name them distinctly; you will be staring at two near-identical properties). Time zone India, currency INR.
4. **Next** through business details and objectives — any answer, they only tune report defaults.
5. **Create**, accept ToS if prompted.

GA4 drops you into "Start collecting data":

6. Platform **Web**. Website URL `https://rishifter.github.io`, stream name `Pages test harness`.
7. **Turn Enhanced measurement OFF.** Left on, GA4 auto-fires scroll, outbound click, file download and more — which floods the `collect` filter you are about to read. You want only the three `test_` events visible.
8. **Create stream.**

The stream detail page shows **Measurement ID `G-XXXXXXX`** top right. Copy it.

GA4 then pushes an "Install your Google tag" panel. **Close it.** GTM does the installing; installing gtag here as well would double every hit.

Repeat the whole thing as `GTM Test – B`. Same account, same URL, Enhanced measurement off again.

**Cross-account makes no difference to collection.** The two measurement IDs can live in entirely separate GA accounts with no shared permissions and the fan-out still works. Account boundaries govern who can *read* data, never whether a hit is accepted — so whatever you test here generalises to the real client scenario.

## 2. GTM container

At <https://tagmanager.google.com>, container `GTM-NW92C2WC`.

First: **Variables → Configure** (top right of the built-in variables box) → under *Utilities*, tick **Event**. The event tags use `{{Event}}` and are dead without it.

### Tags 1 & 2 — the two Google Tags

**Tags → New**, name it `Tag 1`.

1. **Tag Configuration** → *Google Tag*
2. **Tag ID**: `G-AAAAAAA`
3. **Triggering** → *Initialization - All Pages*
4. Save

Repeat identically as `Tag 2` with `G-BBBBBBB`.

Two Google Tags on the page = two configured GA4 destinations. This is the whole mechanism; everything below just routes events into them.

### The shared trigger

**Triggers → New**, name it `Custom Event - test_*`.

1. **Trigger Configuration** → *Custom Event*
2. **Event name**: `^test_`
3. Tick **Use regex matching**
4. Fires on *All Custom Events*
5. Save

### Tags 3 & 4 — the two event tags

**Tags → New**, name it `GA4 Event 1`.

1. **Tag Configuration** → *Google Analytics* → *Google Analytics: GA4 Event*
2. **Measurement ID**: pick `G-AAAAAAA` from the dropdown (it lists the container's Google Tags), or type it
3. **Event Name**: `{{Event}}` — passes through whatever the page pushed
4. **Triggering** → `Custom Event - test_*`
5. Save

Repeat as `GA4 Event 2` with `G-BBBBBBB`, same trigger.

## 3. Preview

**Preview** (top right) → enter `https://rishifter.github.io/test-gtm-ga/` → Connect.

In Tag Assistant's left rail:

| Event in rail | Expect under *Tags Fired* |
| --- | --- |
| `Initialization` | Tag 1, Tag 2 |
| `test_signup` (after a click) | GA4 Event 1, GA4 Event 2 |

Then in the page's own DevTools → Network, filter `collect`: **two hits per click, same `en=test_signup`, different `tid=`.** That is the result.

Publishing is not needed while in Preview. To let someone test independently, **Submit → Publish** — or create a GTM **Environment** for them so Live stays untouched.

## Gotchas

- **Two `page_view` hits on load.** Expected — each Google Tag sends its own. That is the fan-out working. Add config parameter `send_page_view` = `false` on a Google Tag to suppress it.
- **Events silently dropped.** A GA4 Event tag targeting a measurement ID with no matching Google Tag on the page goes nowhere. The *Initialization* trigger is what guarantees both Google Tags run first — do not switch them to *All Pages*.
- **`{{Event}}` renders empty.** The built-in Event variable is not enabled.
- **Enhanced measurement noise.** If the `collect` filter shows events you did not fire, a data stream still has it switched on.
- **GTM will not load over `file://`.** Serve over HTTP.
- **One event name repeating on every hit.** A tag with a hardcoded Event Name reports every event under that name once a catch-all trigger is on it. Useful for isolating one tag in testing, wrong in production. Use `{{Event}}`.
- **Parameters missing from the hit.** `{{Event}}` carries the name only. Each parameter needs a Data Layer Variable and a row on the tag — see [Event parameters](#event-parameters).
- **Hits with an empty `en=`.** The catch-all trigger is matching `gtm.*` events. GA4 rejects them.
- **Parameters from one event appearing on later ones.** The dataLayer never clears a key once pushed. Split the tags or reset the keys — see [Step 2b](#step-2b--clear-the-keys-or-they-leak-onto-later-events).

## Why two event tags rather than one

GTM's GA4 Event tag targets a single measurement ID, so one tag per destination is the supported path.

The one-tag alternative is a Custom HTML tag calling `gtag('event', ...)`, which fans out to every configured destination automatically. Fewer tags — but the events then vanish from GTM Preview's tag list, which is precisely what you are trying to observe. Not worth it here.

## Catch-all trigger

To route *every* dataLayer event rather than just `test_*`:

```
Trigger Configuration → Custom Event
Event name:        .*          [✓] Use regex matching
Fires on:          Some Custom Events
  Condition:       Event  →  does not match RegEx  →  ^gtm\.
```

**The `^gtm\.` exclusion is not optional, including in production.** A bare `.*` also catches `gtm.js`, `gtm.dom` and `gtm.load` on every pageload, plus `gtm.click` / `gtm.scroll` / `gtm.formSubmit` etc. wherever built-in listeners are active. Doubled across two properties that is a lot of requests fired from every user's browser — and they do not even land, because GA4 event names must be letters, numbers and underscores starting with a letter, which `gtm.dom` fails on the dot. Bandwidth spent on hits GA4 was always going to reject.

A negative lookahead in the Event name field (`^(?!gtm\.)`) will not work — GTM's regex engine is RE2, which has no lookahead. Use the condition.

## Event parameters

**A catch-all trigger forwards the event *name*. It does not forward parameters.** `{{Event}}` carries `video_start` through; `video_title`, `video_duration` and the rest are dropped unless each one is declared. Push a parameter-rich event with no declarations and the `collect` hit still fires — just with nothing but `en=video_start` on it.

Verified. One click of `video_start`, two hits:

| `tid=` | `en=` | parameters on the hit |
| --- | --- | --- |
| `G-RD02H7T12Y` | `video_start` | `ep.video_title`, `ep.video_url` |
| `G-PZPWSL5TPH` | `video_start` | *none* |

The button pushes six parameters; only the two declared on Event 1 arrive. `video_duration`, `video_provider`, `video_current_time` and `video_percent` reach neither property — no Data Layer Variable exists for them.

Three things must line up for a parameter to reach a report:

1. A **Data Layer Variable** in GTM, to read the key off the dataLayer.
2. An **Event Parameter** row reaching the GA4 Event tag, mapping that variable to a parameter name.
3. A **custom dimension or metric** in GA4, if the parameter should appear in reports rather than only in DebugView.

Steps 1 and 2 are done once in GTM. Step 3 is per property, so twice.

### Step 1 — Data Layer Variables, one per key

**Variables → User-Defined Variables → New → Data Layer Variable.**

1. **Data Layer Variable Name**: the exact dataLayer key, case-sensitive — `video_title`.
2. **Data Layer Version**: leave at *Version 2*.
3. Leave **Set Default Value** unticked. Unset resolves to `undefined`, which is what makes step 2 skip the row rather than send an empty string.
4. Name the variable `DLV - video_title` (top left) and **Save**.

Repeat for every key the page pushes. For the `video_start` button that is six: `video_title`, `video_url`, `video_provider`, `video_duration`, `video_current_time`, `video_percent`.

This is the one genuinely per-parameter step and there is no way around it — GTM has no "read whatever is on the dataLayer" variable.

### Step 2 — one shared Event Settings variable

Declaring parameter rows on each GA4 Event tag means writing every parameter twice, forever, with the two free to drift. A **Google Tag: Event Settings** variable holds the rows once and both tags inherit them.

**Variables → New → Google Tag: Event Settings**, name it `Event Settings - shared`.

1. **Event Parameters → Add Row.**
2. **Event Parameter**: the GA4 parameter name — `video_title`. Keep it identical to the dataLayer key unless you have a reason not to; two names for one thing is a debugging tax.
3. **Value**: `{{DLV - video_title}}`, picked from the **+** button.
4. Repeat for all six rows. **Save.**

Then attach it to **both** event tags. On each of `GA4 Event 1` and `GA4 Event 2`:

1. **Tags →** the tag **→ Event Parameters** (open the section if collapsed).
2. **Event Settings Variable**: select `Event Settings - shared`.
3. **Save.**

Done once. Every parameter added to the variable later reaches both properties with no tag edit.

**One variable covers every custom event in the container** — but only if you clean up after each event. See the next section; this is the part that bites.

A key that has *never* been pushed resolves to `undefined` and GTM drops its row, so an unused parameter costs nothing. A key pushed *once* is a different matter.

### Step 2b — clear the keys, or they leak onto later events

**GTM's data model is cumulative. `dataLayer.push` merges keys in and never removes them.** Push `video_start` with six parameters and those six Data Layer Variables keep resolving for the rest of the page — so the shared Event Settings variable attaches them to every event that follows.

Verified, one page, in order:

| pushed | video parameters on the hit |
| --- | --- |
| `test_signup`, before any video push | none |
| `video_start` | all six |
| `test_purchase`, after | **all six** |
| `test_cta_click`, after | **all six**, both properties |

Reports then show `video_title` sitting on `test_purchase`, and GA4 gives you no hint that it is stale.

Four ways out, in the order worth considering them.

#### Option A — split the tags by event family *(recommended)*

Makes the leak structurally impossible: a tag that does not fire cannot attach anything. The dataLayer still holds the stale values; they simply have nowhere to go.

Split the one catch-all event tag into a generic one and a video one, per property:

| Tag | Trigger | Event Settings Variable |
| --- | --- | --- |
| `GA4 Event - generic 1` / `2` | catch-all, with `^video_` excluded | none, or a variable holding only universal parameters |
| `GA4 Event - video 1` / `2` | Custom Event, `^video_` | `Event Settings - video` |

1. **Triggers → New → Custom Event.** Event name `^video_`, tick **Use regex matching**, *All Custom Events*. Name it `Custom Event - video_*`.
2. On the existing catch-all trigger, add a second condition alongside the `^gtm\.` one: **Event → does not match RegEx → `^video_`**. Both conditions must hold, which is how GTM treats multiple conditions.
3. **Tags → New → GA4 Event.** Measurement ID `G-RD02H7T12Y`, Event Name `{{Event}}`, **Event Settings Variable** `Event Settings - video`, trigger `Custom Event - video_*`. Name it `GA4 Event - video 1`.
4. Repeat for `G-PZPWSL5TPH` as `GA4 Event - video 2`.
5. Remove the video rows from the shared variable the generic tags use — otherwise they still leak through those.

Cost is two tags per event family rather than two in total. Parameters are still declared once per family and still shared across both properties, so this scales by family, not by parameter. Tags are cheap; a cleanup push that a future developer forgets is not.

**Naming:** property number first, family second — `GA4 Event 2 - video`, `GA4 Event 2 - ecommerce`. That sorts a property's tags together in the GTM list, which matters once there are more than a handful.

**Watch the ordering.** Step 2 stops the catch-all firing on `video_*` for *both* properties, while step 4 gives only the property you have built a tag for somewhere to land. Between the two, any property without its own video tag receives no video events at all. Do step 4 for every property before publishing — or accept the gap knowingly, as this container currently does. See [Current state of the container](#current-state-of-the-container).

#### Option B — reset the keys after each push

One extra push, no container changes. Fine for a small number of parameterised events.

```js
dataLayer.push({
  event: 'video_start',
  video_title: 'Azim Premji University - Campus Tour',
  /* … */
});
dataLayer.push({
  video_title: undefined, video_url: undefined, video_provider: undefined,
  video_duration: undefined, video_current_time: undefined, video_percent: undefined
});
```

Verified: a `test_signup` after that reset carries no video parameters on either property.

The weakness is human, not technical — the reset is easy to forget, and forgetting is silent. Wrap it if you have more than a couple of such events:

```js
function pushEvent(name, params) {
  dataLayer.push({ event: name, ...params });
  dataLayer.push(Object.fromEntries(Object.keys(params).map(k => [k, undefined])));
}
```

If enumerating keys is the objection rather than the reset itself, `dataLayer.push(function() { this.reset() })` clears the entire data model in one line — but it also clears anything else living there, such as `user_id` or consent flags. Safe on a harness, not on a real site.

#### Option C — gate each value on the event name

Keeps a single pair of tags. Each parameter's **Value** becomes a Custom JavaScript variable instead of a plain Data Layer Variable:

**Variables → New → Custom JavaScript**, named `CJS - video_title`:

```js
function() {
  return {{Event}}.indexOf('video_') === 0 ? {{DLV - video_title}} : undefined;
}
```

Returning `undefined` makes GTM drop the row, so the parameter appears only on events whose name starts with `video_`. Point the Event Settings row at `{{CJS - video_title}}` and repeat per parameter.

Untested here. It works in principle but costs one wrapper variable per parameter, each of which looks correct in isolation and is tedious to debug in combination. Prefer A unless you cannot add tags.

#### What does not work

Pushing via gtag's argument syntax, on the theory that its parameters are event-scoped rather than merged:

```js
function gtag() { dataLayer.push(arguments); }
gtag('event', 'video_start', { video_title: '…' });   // ✗
```

Tested: the event fires on both tags and arrives with **no parameters at all**. gtag's arguments syntax puts the values in an `eventModel` that Data Layer Variables cannot read. It avoids the leak by discarding the data.

**Enhanced measurement events are not affected by any of this.** A real outbound click produces `en=click` carrying `link_id`, `link_url`, `link_domain` and `outbound` only. Those events come from the Google Tag and never see your Event Settings variable. If an enhanced-measurement event name shows custom parameters in a report, something is pushing a dataLayer event under the same name and the catch-all trigger is routing it through the GA4 Event tags — GA4 files both under one name.

### Step 3 — register them in GA4

Without this the parameters arrive and are stored, but no report can group by them. DebugView and Realtime show them regardless, which is why a parameter can look fine for a day and be invisible a week later.

**Admin → Data display → Custom definitions.** Per property, so do this in both.

**Text parameters → Custom dimensions tab → Create custom dimension:**

1. **Dimension name**: what appears in reports — `Video title`.
2. **Scope**: *Event*.
3. **Event parameter**: the exact parameter name from step 2 — `video_title`.
4. **Save.**

**Numeric parameters → Custom metrics tab → Create custom metric:** same fields, plus **Unit of measurement** — *Standard* for a count or percentage, *Seconds* for a duration. `video_duration` and `video_current_time` are seconds; `video_percent` is standard.

Getting this split wrong is the common mistake: a number registered as a dimension gives you rows of distinct values rather than something you can sum or average.

Caps are **50 event-scoped custom dimensions** and **50 custom metrics** per property, so budget them rather than registering every parameter reflexively. Only register what a report needs to slice by.

Data is **not** backfilled — a dimension only populates from hits received after it is created. Reports typically take 24–48 hours to show values.

> Uncertain: `video_title`, `video_url`, `video_provider`, `video_duration`, `video_current_time` and `video_percent` are GA4's own recommended parameters for video events, and GA4 populates its Video engagement report from them when enhanced measurement generates the events. Whether the same parameters sent manually from a GA4 Event tag are picked up natively, or still need registering as custom definitions, is not confirmed. Register them and find out — an unnecessary custom dimension costs one of the fifty, nothing worse.

### Verifying the parameters

1. **Network → `collect`** — the parameters ride on the hit as `ep.<name>` for text and `epn.<name>` for numbers. Expect `ep.video_title=Azim%20Premji…` and `epn.video_duration=180` on **both** `tid=` values. This step needs GTM only; it works before anything is registered in GA4.
2. **GTM Preview** — click the tag under *Tags Fired*, check the parameter rows resolved to values rather than `undefined`. A row showing `undefined` means the dataLayer key and the Data Layer Variable name disagree, usually on case.
3. **GA4 DebugView** — confirms GA4 accepted them.
4. **Reports** — 24–48 hours after registering, the dimension appears in explorations.

> The parameter-dropping behaviour is verified. The shared **Event Settings variable** is not — the container uses inline per-tag rows, so "both tags inherit the same rows" still needs a run. See [§2 Shared Event Settings variable](#2-shared-event-settings-variable).

### Zero-maintenance alternative

If per-parameter config is unacceptable at scale, a Custom HTML tag forwarding the whole dataLayer object costs nothing per parameter and fans out to every configured destination on its own — at the price of GTM Preview visibility. See [§3 Custom HTML `gtag('event', ...)`](#3-custom-html-gtagevent-).

## Current state of the container

Option A is rolled out on **property 2 only**. The two properties are deliberately not symmetrical — Anita is working on Event 1, Rishi on Event 2, so the two structures can be compared side by side under the same traffic.

| | `G-RD02H7T12Y` | `G-PZPWSL5TPH` |
| --- | --- | --- |
| Owner | Anita | Rishi |
| Generic tag | `GA4 Event 1` | `GA4 Event 2` |
| Video tag | none | `GA4 Event 2 - video` |
| Parameters on the generic tag | `Event Settings - video`, plus inline `video_title` and `video_url` | none |
| Structure | pre-Option A | Option A |

Naming: the video tag is **`GA4 Event 2 - video`** — property number first, family second. `GA4 Event 2 - video`, `GA4 Event 2 - ecommerce`, and so on, sorts all of a property's tags together in the GTM list. Use the same shape for property 1 when it gets one.

Triggers, as published: the catch-all is `.*` on *Some Custom Events*, excluding both `^gtm\.` and `^video_`; the video trigger is `^video_` on *All Custom Events*.

Three consequences to expect while reading `collect` traffic, so they do not read as faults:

- **`video_start` no longer reaches `G-RD02H7T12Y` at all.** The catch-all excludes `^video_` for both properties, but only property 2 has a video tag to pick those events up. Property 1 gets video events again when it gets its own `GA4 Event 1 - video`, or sooner by narrowing the exclusion.
- **Property 1 still leaks video parameters onto other events**, from its Event Settings Variable and its two leftover inline rows. Combined with the point above, that property currently sees `video_title` on `test_cta_click` while seeing no `video_start` whatsoever. Expected for now; both go away with step 5 of Option A.
- **Property 2 needs no reset push.** A `test_cta_click` after a `video_start` carries no video parameters, because the tag that would attach them does not fire on it. This is the difference Option A buys.

Enhanced measurement is still on for at least one stream, which is where the stray `page_view` and `scroll` hits come from.

---

# Other approaches, not yet tested

Alternatives to the two-Google-Tags + two-event-tags pattern. **None of these have been verified in this harness.** Recorded so the next person does not re-derive them.

## The problem they address

Two catch-all GA4 Event tags scale fine for event *names* — two tags total, however many events you push. What does not scale is **parameters**: a GA4 Event tag does not auto-forward arbitrary dataLayer keys, so each parameter is declared explicitly, in both tags, forever, with a standing risk they drift apart and the two properties quietly disagree.

## 1. GA4 Destinations (formerly Connected Site Tags)

Configure the fan-out in GA4 rather than GTM. One Google Tag on the page; GA4 forwards to the additional properties.

**Admin → Data Streams → your stream → Configure tag settings → your Google Tag → + Destination → Choose destination.**

- Events without a `send_to` parameter go to the `default` target group — every configured destination, parameters included, no per-property mapping.
- `send_to` is available if specific events should reach only specific properties.
- **You can only connect properties you own.** If the scenario is "our property plus the client's own property in their account", this is unavailable.
- **Connecting a property as a destination removes it from any other Google tag that previously had it.** Silent side effect on whatever else was using it.

**Open question:** [one walkthrough](https://datajournal.datakyu.co/ga4-connected-site-tags/) states destinations require a gtag.js install because gtag.js is not loaded via GTM. That is probably wrong — GTM's Google Tag template fetches its config from Google's servers by tag ID, and destinations are part of that server-side config — but no Google documentation confirms it either way.

**Experiment that settles it**, using this harness: configure Property B as a Destination on Property A in GA4, then delete `Tag 2` and `GA4 Event 2` from GTM so only Property A's tag remains. Click a button and read the `collect` filter.

- Two `tid=` values → destinations work through GTM, and the two-tag pattern is unnecessary.
- One `tid=` → the article is right, GTM-deployed tags ignore destinations.

## 2. Shared Event Settings variable

Both GA4 Event tags reference a single **Google Tag Settings / Event Settings** variable. Parameters are declared once and both tags inherit them. Keeps the two-tag structure and its Preview visibility, but removes most of the drift risk.

## 3. Custom HTML `gtag('event', ...)`

Fans out to every configured destination automatically, parameters included, with no per-property event tag. Its drawback — invisibility in GTM Preview — matters far less in production than in a harness built specifically to observe tag firing.

## Reference

- [Google tag routing and `send_to`](https://developers.google.com/tag-platform/gtagjs/routing)
- [GA4 connected site tags walkthrough](https://datajournal.datakyu.co/ga4-connected-site-tags/)
