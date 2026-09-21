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

Replace `GTM-XXXXXXX` in `index.html` with your container ID, then build four tags:

| Tag | Type | Setting | Trigger |
| --- | --- | --- | --- |
| Google Tag A | Google Tag | `G-AAAAAAA` | Initialization – All Pages |
| Google Tag B | Google Tag | `G-BBBBBBB` | Initialization – All Pages |
| Event → A | GA4 Event | Measurement ID `G-AAAAAAA`, event name `{{Event}}` | Custom Event, regex `^test_` |
| Event → B | GA4 Event | Measurement ID `G-BBBBBBB`, event name `{{Event}}` | same trigger |

Two Google Tags = two configured destinations. GA4 Event tags target a single measurement ID each, so one per destination is the supported path.

You can point both Google Tags at **fake** measurement IDs and still see the two hits — the routing is observable without owning either property.

## Verifying

1. **Network tab** — the actual test. Two `collect` hits per click.
2. **GTM Preview** (Tag Assistant → `http://localhost:8000` or the Pages URL) — shows which tags fired and why.
3. **GA4 DebugView** — only if you want to confirm data lands. Tag Assistant sets `debug_mode` for you.

Layer 1 is the test; 2 and 3 are confirmation.

## Not covered

Consent Mode, server-side GTM, event deduplication. Add consent handling before reusing any of this for an EU-facing client.
