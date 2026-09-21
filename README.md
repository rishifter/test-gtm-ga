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
