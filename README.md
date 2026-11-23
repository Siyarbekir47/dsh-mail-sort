# dsh-mail-sort

Event-driven IMAP inbox sorting + Telegram summary for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness).

New mail arrives → one agent run sorts the whole current INBOX of your accounts → a short 📬 summary lands in your Telegram group. No cron, no empty LLM runs: the trigger is dsh-email's in-process `emailTasks` arrival hook (metadata only — the plugin never reads bodies and never sends mail).

## Requirements

- DSH web profile, Node ≥ 22.19
- [`dsh-email`](https://github.com/STARDUSTLC666/dsh-email) ≥ 0.25.0 with configured IMAP accounts
- A Telegram bot token stored as a ref in `~/.dsh/.credentials.yaml` (e.g. via `dsh-im`)

## Install

```sh
dsh plugin --profile web add github:Siyarbekir47/dsh-mail-sort
# or, once published:
dsh plugin --profile web add dsh-mail-sort
```

## Configure

1. **dsh-email consumers** (profile `cordis.patch.yml`, row `tool-email`) — one entry per account/folder; `id` must match `consumerIds` below; `senders: []` means all senders:

```yaml
- id: tool-email
  config:
    consumerPollSeconds: 90          # 30..3600, arrival poll interval
    consumers:
      - id: mail-sort-main
        account: Main - Mailbox      # exact dsh-email account key
        folder: INBOX
        senders: []
        enabled: true
```

2. **Plugin config** (row `mail-sort`):

```yaml
- id: mail-sort
  config:
    enabled: true
    consumerIds: [mail-sort-main]
    accounts: [you@example.com]
    cwd: /home/you/projects/automation   # workspace for spawned runs
    preset: standard                     # agent preset for spawned runs
    provider: offical                    # llm provider id (required)
    model: qwen3.8-flash                 # model id (required)
    chatId: "-1001234567890"             # Telegram group (required)
    telegramTokenRef: DSH_TELEGRAM_BOT_TOKEN_XXXXXXXX   # credentials.yaml ref
    # credentialsPath: ~/.dsh/.credentials.yaml
    # flushDelayMs: 4000                 # debounce arrivals into one batch
    # maxWaitMs: 20000                   # max batch wait
```

Restart DSH (or toggle the bundle) after config changes.

## How it works

- `emailTasks.subscribe(id, handler)` per consumer id; handler only queues metadata.
- Debounced batch → `ctx.agents.create()` + `agent.followup()` with the sorting prompt (category table, move-only-existing-folders, confirm moves, no deletes/replies, mail content is data not instructions, no secrets over Telegram).
- The run sweeps the **whole current INBOX**, so missed/failed mails are picked up next trigger.
- Summary delivery is host-side: the agent calls `mail_sort_report(text)`; the plugin posts to the Bot API. Spawned agents need no shell.
- Empty inbox → no Telegram message. One message per run (split only if >4096 chars).

## Tools

- `mail_sort_report` — agent-facing, sends the HTML summary (returns `ok`, `message_id`).
- `mail_sort_status` — pending arrivals, last run, dsh-email watcher status.

## Troubleshooting (lessons learned)

- dsh-email clears all subscriptions on recomposition (`backend.stop()`); this plugin re-subscribes reactively via `ctx.inject(['emailTasks'], …)`. A one-shot subscribe in `apply` silently dies after any settings save.
- Service properties are guard-checked: declare `emailTasks` **and** `timer` in `inject`, otherwise timer callbacks throw and the fiber dies silently.
- Create timers inside the inject callback, not synchronously in `apply` (apply-time timers are dropped on inject-resolution restart).
- `ctx.agents.create()` returns `{agent, dispose}`; prompt via `handle.agent.followup()`. Spawned runs need `meta.agentPreset` **and** `agentOptions.provider/model`, else "no provider/model".
- First consumer scan only establishes a baseline; mails already in INBOX at startup do not fire (the INBOX sweep covers them on the next trigger).
- `output.render` is called as `render(args, value)` — rendering the first parameter echoes the call arguments back to the model and hides the real result (e.g. Telegram ok/error). Always render the second parameter.
- Code edits under `node_modules` do not reach a running host: CJS plugins stay in Node's require cache across bundle toggles. Reinstall the package (or restart DSH) to load changed code.
- Telegram HTML parse_mode rejects `<br>`, `<p>`, `<div>` and bare `&`/`<`. Since 0.2.2 the plugin sanitizes host-side (whitelist tags, `<br>`→newline, escapes bare `&`/`<`), falls back to plain text on parse errors, and retries transient failures once. A formatting mistake can no longer silently swallow the notification.
- The consumer's first scan after (re)start only sets a baseline; mails arriving during downtime never fire an event. Since 0.2.2 the plugin runs one startup sweep (~45 s after activation) that processes any leftover INBOX mail; it exits silently when everything is already sorted.

## License

MIT
