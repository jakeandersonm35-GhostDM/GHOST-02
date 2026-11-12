# Ghost: setup guide

Two parts:

| Part | Where it runs | What it holds |
|---|---|---|
| **ghost-core/** | Your Huawei Cloud server | **All API keys**, the brains, memory, the approval queue, social/email/WhatsApp connectors |
| **app/** | Your phone | Voice, the HUD, phone control. Stores only the server address and one login token. |

Brains, in the order Ghost uses them:

```
Phone --> Ghost Core --> OmniRoute (optional gateway)
                          |
   MAIN: Groq   --> ASSISTANTS: OpenAI x3 (writer, coder, analyst)
                          |
          Free-LLM, Agentspace (extra fallbacks)
                          |
   internet/brains down --> OFFLINE: Ollama --> CAMEL role-play
```

## 1. Create the server on Huawei Cloud
1. Create an **ECS** instance: Ubuntu 22.04, **4 vCPU / 8 GB RAM** (the offline Ollama brain needs the RAM), 40 GB disk.
2. **Pick a region outside mainland China** (e.g. Hong Kong or Singapore). Groq and OpenAI are generally not reachable from mainland regions, and a domain there needs ICP filing.
3. Security group: allow inbound **22, 80, 443**.
4. Point a domain (A record) at the server's public IP, e.g. `ghost.yourdomain.com`. Edit `ghost-core/Caddyfile` with that name. This gives you HTTPS, which the token needs.
5. On the server:
   ```bash
   curl -fsSL https://get.docker.com | sh
   # copy the ghost-core folder here (scp, or Huawei's file upload), then:
   cd ghost-core
   cp .env.example .env
   nano .env          # fill in keys (step 2 below)
   docker compose up -d --build
   docker compose exec ollama ollama pull llama3.2     # offline brain, one time
   curl https://ghost.yourdomain.com/health            # should print {"ok":true,...}
   ```

## 2. Add the API keys (on the server, in `.env`, never in the app)
Start with only the first block; Ghost tells you in the app what is still missing.

| Block | Keys | Where to get it |
|---|---|---|
| **Required** | `GHOST_TOKEN` (run `openssl rand -hex 24`), `PUBLIC_URL`, `OWNER_NAME` | you choose |
| Main brain | `GROQ_API_KEY` | console.groq.com |
| 3 assistants | `OPENAI_API_KEY` (1 key is enough) | platform.openai.com |
| Gateways | `OMNIROUTE_BASE_URL` (+key), `FREELLM_*`, `AGENTSPACE_*` | your OmniRoute install (default `http://host:20128/v1`) and the provider endpoints you use; all must be OpenAI-compatible |
| Memory | `HONCHO_API_KEY` | app.honcho.dev |
| YouTube + Gmail | `GOOGLE_CLIENT_ID/SECRET` | Google Cloud Console, OAuth client type *Web*; redirect URI `PUBLIC_URL/oauth/callback/google`; enable YouTube Data, YouTube Analytics, Gmail APIs |
| Facebook + Instagram | `META_APP_ID/SECRET` | developers.facebook.com; redirect URI `PUBLIC_URL/oauth/callback/meta`. Instagram must be a Professional account linked to your Facebook Page |
| WhatsApp | `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_TOKEN` | Meta WhatsApp Business Cloud API |
| Own phone number + calls | `TWILIO_ACCOUNT_SID/AUTH_TOKEN` | twilio.com |
| Business income | `STRIPE_API_KEY` and/or `PAYSTACK_SECRET_KEY` | your dashboard |

After editing: `docker compose up -d` again.

## 3. Build and install the app
Follow `app/BUILD-APK-INSTRUCTIONS.txt` (GitHub builds the APK for you) and `app/INSTALL-ON-HUAWEI-NOVA-12i.md` for the phone settings (background running, accessibility).

## 4. Press START GHOST
Open the app, enter the server address and `GHOST_TOKEN`, press **START GHOST**. The app then:
- tells the server to go live and shows which keys are still missing,
- points all of its AI calls at your server,
- starts a notification-backed background service so Ghost keeps running with the screen locked.

Then **Connections** (menu) to link Gmail, YouTube, Facebook, Instagram. **Ghost Control** (menu) shows approvals.

## How consent works
Ghost can only *propose* posting, emailing, WhatsApp, calling, buying a number, running code, or adding/removing agents and skills. Each proposal appears as a card with the exact content. Nothing happens until you tap **Approve**, and the server runs exactly the text you saw.

## Things to know (honest limits)
- **Not compiled or run on a phone by me.** The server passed an automated 14-check test (routing, fallback to offline, approvals gate). The Flutter/Kotlin changes were written against your source but need a real build; if GitHub's build turns red, send the last 40 log lines.
- **Voice:** your zip contains no audio recording of Jarvis. It speaks with Microsoft's `en-GB-RyanNeural` neural voice (already the default there), which is a British butler-style voice, not a clone of the film actor. Voice cloning someone's voice is not something I'd set up.
- **Locked phone:** the keep-alive service helps, but Huawei may still kill background apps; do the battery settings in the install guide. Microphone listening while locked depends on the phone's speech service.
- **Phone with no internet:** it cannot reach your Huawei server. For that case the app falls back to an Ollama running on the phone itself (e.g. in Termux, `127.0.0.1:11434`) or on your home network.
- **Own number:** Twilio can buy you a number for calls/SMS (costs money). A WhatsApp number must be registered with Meta; it cannot be auto-generated.
- **Income:** YouTube estimated revenue (monetised channels), Stripe/Paystack, and a manual ledger work. Instagram has no earnings API; Facebook earnings depend on what Meta exposes for your Page.
- **Honcho** calls follow its v3 docs but I could not test against a live account; if memory does not appear, Ghost still keeps a local memory file.
- **CAMEL** is implemented natively (two role-playing agents on Ollama), not via the camel-ai package. **Hermes Agent** (Nous Research) is a separate install; the app's own Hermes-style engine (SOUL/USER/AGENT notes) is what learns about you. Ghost's "add agents and skills" are stored configurations, not self-editing source code.
- `ALLOW_CODE_RUN=1` lets approved code run on the server in an empty-environment child process. That is not a full sandbox. Leave it `0` unless you need it.
