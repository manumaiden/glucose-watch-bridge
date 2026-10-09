# Setup guide

The order matters: storage and delivery first, then the workflow that feeds them.

1. [LibreLinkUp follower account](#1-librelinkup-follower-account)
2. [ntfy](#2-ntfy)
3. [Nightscout](#3-nightscout)
4. [n8n workflows](#4-n8n-workflows)
5. [Phone and watch](#5-phone-and-watch)
6. [Optional: xDrip+](#6-optional-xdrip)
7. [Troubleshooting](#7-troubleshooting)

Placeholders used below:

| Placeholder | Example |
|---|---|
| `<NTFY_HOST>` | `ntfy.example.com` or `192.168.1.10:8082` |
| `<NTFY_TOPIC>` | `glucose` (anything hard to guess is fine) |
| `<NIGHTSCOUT_HOST>` | `192.168.1.10` |

---

## 1. LibreLinkUp follower account

1. On the **patient's** phone, open **LibreLink** → *Connected Apps* → *LibreLinkUp* → add a connection and enter the follower's email address.
2. Install **LibreLinkUp** (the follower app), sign up with that email and accept the invitation.
3. Open LibreLinkUp once and accept all the prompts (terms of use, privacy).

These follower credentials are the ones n8n will use. You only need the app itself again when Abbott asks to re-accept new terms: the error workflow will tell you.

## 2. ntfy

```bash
cd docker/ntfy
cp config/server.yml.example config/server.yml   # set base-url
docker compose up -d

# create a user and give it access to your topic
docker exec -it ntfy ntfy user add <NTFY_USER>
docker exec ntfy ntfy access <NTFY_USER> <NTFY_TOPIC> rw
```

With `auth-default-access: deny-all`, anonymous requests get `403`. Quick test:

```bash
curl -u <NTFY_USER>:<password> -d "test" http://<NTFY_HOST>/<NTFY_TOPIC>
```

To receive alerts away from home, expose ntfy over HTTPS (reverse proxy, tunnel or VPN, see [requirements](requirements.md#network)).

## 3. Nightscout

```bash
cd docker/nightscout
cp .env.example .env        # set NIGHTSCOUT_API_SECRET and NIGHTSCOUT_BASE_URL
# pick the right mongo tag for your CPU (comments in docker-compose.yml)
docker compose up -d
```

Open `http://<NIGHTSCOUT_HOST>:1337` and complete the first-run profile. n8n authenticates with the **SHA1** of the API secret:

```bash
echo -n "<your NIGHTSCOUT_API_SECRET>" | sha1sum
```

## 4. n8n workflows

### 4.1 Create the credentials

In n8n → *Credentials* → *Add credential*:

| Name (suggested) | Type | Values |
|---|---|---|
| `LibreLinkUp login` | **Custom Auth** | `{"body": {"email": "<follower email>", "password": "<follower password>"}}` |
| `Nightscout API secret` | **Header Auth** | Name: `api-secret`, Value: the SHA1 from step 3 |
| `ntfy` | **Basic Auth** | the ntfy user and password from step 2 |

### 4.2 Import the workflows

1. *Workflows* → *Import from file* → `n8n/glucose-watch-bridge-error-alert.json`
2. *Import from file* → `n8n/glucose-watch-bridge.json`
3. In every HTTP node with a credential, select the matching credential from 4.1. The imported files reference a dummy `REPLACE_WITH_YOUR_CREDENTIAL` id.

### 4.3 Replace the placeholders

| Node | Field | Replace |
|---|---|---|
| `POST to Nightscout` | URL | `<NIGHTSCOUT_HOST>` |
| `ntfy Send` (main) and `ntfy Alert` (error workflow) | URL | `<NTFY_HOST>` and `<NTFY_TOPIC>` |
| `LLU Login`, `LLU Get Connections` | URL | only if you are not in the EU: `api-eu` → your region (`api-us`, `api-de`, `api-fr`, `api-au`, …). The login error tells you the right one |
| `Compute account-id` | `accountId` | see below |

**account-id.** LibreLinkUp requires an `account-id` header equal to the SHA256 of your follower `user.id`. The n8n sandbox doesn't allow the `crypto` module, so the hash is computed once by hand:

1. Run the main workflow manually once. `Compute account-id` fails with a message like
   `account-id not configured. Your user.id is 1a2b3c… — run: echo -n "1a2b3c…" | sha256sum`
2. Run that command and paste the 64-character hash into `accountId`.

### 4.4 Connect the error workflow and activate

1. Main workflow → *Settings* → **Error workflow** → select *Glucose Watch Bridge — Error alert*.
2. Adjust the [thresholds](thresholds.md) in `Prepare ntfy`.
3. Activate (publish) **both** workflows.

After 5 minutes you should see a new entry in Nightscout and, within 15 minutes, a notification on the phone.

## 5. Phone and watch

1. Install the **ntfy** app (Play Store or F-Droid), add your server `https://<NTFY_HOST>` with the user from step 2 and subscribe to `<NTFY_TOPIC>`.
2. In Android settings, **disable battery optimization** for ntfy. On MIUI/HyperOS also enable **Autostart**. Otherwise notifications arrive late or not at all when the phone is idle.
3. In the watch companion app (e.g. Huawei Health → *Devices* → *Notifications*), **enable notifications for ntfy**.
4. Urgent alerts use ntfy priority `urgent`. If you want them to vibrate on the watch, check that the ntfy notification channels for high/max priority aren't silenced on the phone.

## 6. Optional: xDrip+

For charts on the phone, install [xDrip+](https://github.com/NightscoutFoundation/xDrip/releases) (APK from GitHub) and set:

- *Hardware Data Source*: **Nightscout Follower**
- *Nightscout Follow URL*: `https://<API_SECRET>@<NIGHTSCOUT_HOST>/api/v1/`. The **trailing slash is required**, otherwise: `IllegalArgumentException: baseUrl must end in /`.

Disable battery optimization for xDrip+ too.

## 7. Troubleshooting

| Symptom | Where to look |
|---|---|
| No notifications at all | Is there a recent entry in Nightscout (`/api/v1/entries.json?count=1`)? If not, the problem is in n8n: open the failed executions |
| `⚠️ Glucose pipeline DOWN` alert | The message names the failing node and the reason. Most common: *action required "tou"*, so open LibreLinkUp and accept the new terms |
| Entries in Nightscout but no notification | Check the ntfy credential and URL. Then check battery optimization and that the topic is subscribed on the phone |
| Notification on the phone but not on the watch | Enable notifications for the ntfy app in the watch companion app |
| `401` from Nightscout | The Header Auth credential must contain the **SHA1** of the API secret, not the secret itself |

More known issues are listed in the [README](../README.md#lessons-learned-gotchas).
