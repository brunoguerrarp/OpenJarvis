# Telegram bot in the cloud (phone assistant)

Run OpenJarvis as an always-on **Telegram bot** so you can talk to your personal
AI from your phone — anywhere, without keeping your own computer on. The server
and a cloud model run in the cloud; you chat with it from the Telegram app.

```mermaid
graph LR
    A[Telegram app on your phone] -->|message| B[Telegram Bot API]
    B -->|long polling| C[OpenJarvis server in the cloud]
    C -->|cloud engine| D[gpt-4o-mini]
    D --> C --> B --> A
```

This path needs no new code: OpenJarvis already ships a Telegram channel and a
cloud engine. You wire them together with one config file and three secrets.

## What you need

- A **Telegram bot token** — free, from [@BotFather](https://t.me/BotFather).
- A **cloud model key** — e.g. an `OPENAI_API_KEY`.
- A **host that stays online.** Telegram delivery here uses *long polling*, so
  the process must run continuously. A Render **free** service spins down when
  idle and the bot goes silent, so use an always-on plan (or your own always-on
  machine — see *Local / own-machine* below).

## 1. Create the bot

1. In Telegram, open [@BotFather](https://t.me/BotFather) and send `/newbot`.
2. Pick a name and a username ending in `bot`.
3. Copy the **token** it gives you (looks like `123456:ABC-DEF...`).
4. Message [@userinfobot](https://t.me/userinfobot) and copy **your numeric
   chat ID** — you'll use it to lock the bot to yourself.

## 2. Lock the bot to your chat

Edit `configs/openjarvis/examples/telegram-cloud.toml` and set your ID so
strangers who find the bot can't spend your model budget:

```toml
[channel.telegram]
allowed_chat_ids = "123456789"   # your ID from @userinfobot
```

Leaving it empty lets anyone who discovers the bot talk to it — not recommended.

## 3a. Deploy on Render (always-on, recommended)

A ready blueprint lives at `deploy/render-telegram.yaml`.

1. Push your edited repo (with your `allowed_chat_ids`) to your own GitHub fork.
2. In Render, start a **New Blueprint Instance** and select that fork.
   Point it at `deploy/render-telegram.yaml` if asked for the blueprint path.
3. When Render prompts for secrets, provide:
   - `TELEGRAM_BOT_TOKEN` — the @BotFather token
   - `OPENAI_API_KEY` — your cloud model key
4. Apply and wait for `/health` to go healthy.
5. Open your bot in Telegram and say hi.

The blueprint uses the always-on `starter` plan, pins `PORT`/`--port` to
`10000`, generates `OPENJARVIS_API_KEY` (the server refuses a public bind
without one), and enables the channel via `OPENJARVIS_CONFIG` pointing at the
preset baked into the image.

!!! warning "Ephemeral filesystem"
    Like the base Render blueprint, state under `/home/openjarvis/.openjarvis`
    is not durable across restarts/redeploys. Fine for a stateless chat bot;
    for memory or connector tokens, attach a persistent disk — see
    [Deploy on Render](render.md).

## 3b. Local / own-machine (free)

If you'd rather keep the brain on a machine you already run 24/7 (a home
server, a mini PC, a cheap VPS):

```bash
export TELEGRAM_BOT_TOKEN="123456:ABC..."
export OPENAI_API_KEY="sk-..."
export OPENJARVIS_API_KEY="$(jarvis auth generate-key)"
OPENJARVIS_CONFIG=configs/openjarvis/examples/telegram-cloud.toml \
  jarvis serve --host 0.0.0.0 --port 8000
```

As long as that process stays running, your bot answers from your phone. No
inbound ports or tunnel are needed — the server reaches *out* to Telegram.

## Choosing the model and agent

The preset uses the single-turn `simple` agent and `gpt-4o-mini` for cheap, fast
replies. To give the bot tools (web search, code, files), set
`default_agent = "orchestrator"` under `[channel]` in the preset. To change the
model, edit `--model` in the blueprint (and `default_model` in the preset).

## Bot commands

Once connected, the channel understands a few commands in chat: `/help`,
`/agents`, `/agent <id> status`, `/sessions`, `/more`. See
[Channels](../user-guide/channels.md) for the full list and for swapping
Telegram for WhatsApp, Signal, Discord, and others.
