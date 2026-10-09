# Alert thresholds

Every 5 minutes the workflow decides whether to send a notification and with which priority. The decision happens in the **`Prepare ntfy`** node of `n8n/glucose-watch-bridge.json`.

## The settings

At the top of the `Prepare ntfy` Code node:

```js
const LOW_MMOL = 3.9;          // below → urgent "LOW" alert      (≈ 70 mg/dL)
const HIGH_MMOL = 10.0;        // above → urgent "HIGH" alert     (≈ 180 mg/dL)
const USE_LIBRE_FLAGS = true;  // also alert when LibreLinkUp itself flags isHigh/isLow
const NORMAL_EVERY_MIN = 15;   // silent "normal" notification every N minutes (multiple of 5)
```

| Setting | Effect |
|---|---|
| `LOW_MMOL` | A reading **below** this value sends `🔴 LOW GLUCOSE` with ntfy priority `urgent` (the watch vibrates). Set it to `null` to disable |
| `HIGH_MMOL` | A reading **above** this value sends `🟡 HIGH GLUCOSE` with priority `urgent`. Set it to `null` to disable |
| `USE_LIBRE_FLAGS` | When `true`, the `isLow` / `isHigh` flags returned by LibreLinkUp also trigger an urgent alert, even if the value is inside your thresholds |
| `NORMAL_EVERY_MIN` | Readings inside the range are sent as silent `🩸 Glucose` notifications only once every N minutes (`15` = 4 per hour, `5` = every reading, `60` = once per hour) |

Urgent alerts are **never throttled**: while the value stays out of range, you get one every 5 minutes.

## Units

The thresholds are in **mmol/L**. To convert from mg/dL, divide by 18:

| mg/dL | mmol/L |
|---|---|
| 54 | 3.0 |
| 70 | 3.9 |
| 180 | 10.0 |
| 250 | 13.9 |

So for `LOW < 70 mg/dL` and `HIGH > 250 mg/dL`, use `LOW_MMOL = 3.9` and `HIGH_MMOL = 13.9`.

## Two sources of "out of range"

1. **The workflow thresholds** (`LOW_MMOL` / `HIGH_MMOL`) are under your control and live in n8n. Changing them takes effect at the next run, no restart needed: save the workflow.
2. **The LibreLinkUp flags** (`isHigh` / `isLow`) come from the alarm settings configured in the **patient's LibreLink app**. The follower can't change them from n8n. With `USE_LIBRE_FLAGS = false` they are ignored.

If you only want to rely on the patient's app alarms, set both thresholds to `null` and keep `USE_LIBRE_FLAGS = true`.

## Nightscout ranges are separate

`BG_LOW`, `BG_TARGET_BOTTOM`, `BG_TARGET_TOP` and `BG_HIGH` in `docker/nightscout/docker-compose.yml` only affect the **Nightscout dashboard** (chart colours and its own browser alarms). They do **not** change the ntfy notifications. Keep them consistent with your thresholds to avoid confusion.

## Changing the notification text

In the same node:

- `title`: `🔴 LOW GLUCOSE`, `🟡 HIGH GLUCOSE`, `🩸 Glucose`. Translate them if you like (the screenshots in the README use an Italian title).
- `body`: `6.9 mmol/L (125 mg/dL) ↗`. The trend arrows map the LibreLinkUp `TrendArrow` values 1–7.
