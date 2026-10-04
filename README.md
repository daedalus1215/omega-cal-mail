# omega-cal-mail

**calmail** turns Proton Mail subject lines into calendar events, hands-free.

Email yourself a subject starting with `cal:` from your phone and the event
lands in your shared omega calendar — no terminal, no paste, no box login.
A small daemon watches your Proton inbox, hands each request to a headless
AI session (`omp` + the `omega-cal` skill) that does all the
natural-language parsing, writes the event through the omega calendar API,
and replies to the sender with a one-line confirmation.

```
you (phone) --"cal: tues 7:20am pickup cc"--> Proton inbox
                                                 |
                    calmail (systemd user service)
                                                 |
                     headless `omp -p` session + omega-cal skill
                                                 |
                     event in the omega shared calendar
                                                 |
          reply email to the sender: "✅ Pick up cc Tue 7:20–8:20am (#6003)"
```

## How it works

- A Python daemon (`calmail.py`, built on `protonmail-api-client`) keeps a
  Proton session alive and waits for new mail via Proton's event-polling
  API; a direct inbox scan cross-checks every ~5 minutes as a safety net
  (the feed can go healthy yet silent), repeated failures degrade to pure
  inbox polling, and every API request is capped at 30 s so a stalled
  connection cannot freeze the loop.
- A message is a calendar request when its **subject** starts with the
  prefix (`cal:`, case-insensitive). The text after the prefix is the
  request; a non-empty body is appended to it (HTML is stripped to text).
- The combined text is fed to a headless `omp -p` session that has the
  `omega-cal` skill installed. The skill owns all parsing — terse one-offs,
  weekly blocks, "exact same", am/pm inference, move/cancel — and talks to
  the omega API with the bot account it is configured with. Production is
  the default unless the request names `dev`.
- The final line of the session becomes a **threaded reply to the sender**,
  so you (or whoever forwarded the request) get the ✅/❌ confirmation.
- Processed message ids are remembered in `state.json`; a catch-up scan at
  startup handles mail that arrived while the machine was off.

## Requirements

- Linux with systemd user sessions
- Python 3.11+ and [`uv`](https://docs.astral.sh/uv/)
- A Proton Mail account (any address on it works as the mail target)
- The `omp` CLI with a headless print mode (`omp -p`)
- The `omega-cal` skill installed for omp (`~/.agents/skills/omega-cal/`),
  including its omega API bot credentials
  (`~/.config/omega/assistant.env`)

## Setup (one-time, ~5 min)

```bash
cd omega-cal-mail
uv sync                                            # creates .venv

mkdir -p ~/.config/omega-cal-mail
cp env.example ~/.config/omega-cal-mail/env
chmod 600 ~/.config/omega-cal-mail/env
$EDITOR ~/.config/omega-cal-mail/env                # your Proton creds

.venv/bin/python calmail.py login                  # first login (may auto-solve a CAPTCHA)
.venv/bin/python calmail.py check                  # verify the saved session

cp calmail.service ~/.config/systemd/user/calmail.service
systemctl --user daemon-reload
systemctl --user enable --now calmail
journalctl --user -u calmail -f                     # watch it work
```

If the first login hits a CAPTCHA the solver retries a few times; if it
still fails, re-run `calmail.py login` in a terminal — a fresh puzzle is
often easier.

## Usage

From any mail app, email any address on your Proton account (usually
yourself):

| Subject | Meaning |
|---|---|
| `cal: tues 7:20am pickup cc` | one-off event |
| `cal: fri 7:20am. tues 3:30pm.` | several one-offs |
| `cal: week of oct 12` *(block in the body)* | weekly block, one event per (day, time) |
| `cal: move dentist to 2pm` | move an existing event |
| `cal: cancel dentist` | cancel an existing event |
| `cal: what's on tues` | list (replied inline) |

Weekly block body format (handled by the skill):

```
Week of Oct 12:
Tuesday: drop off Ella 7:30, cece 8:30, pick up Cece 3:30
Thursday: exact same
Friday: none
```

You get a threaded reply like `✅ "Pick up cc" Tue 7:20–8:20am (#6003)`,
or for a block a summary line
(`✅ created 15 events, week of Oct 12 (#5990–#6004)`).

Optional polish: an iOS Shortcut / Android quick tile that pre-fills a mail
to yourself with subject `cal: ` makes it a two-tap action.

## Operations

| What | Where |
|---|---|
| logs | `journalctl --user -u calmail -f` |
| config | `~/.config/omega-cal-mail/env` (edit, then `systemctl --user restart calmail`) |
| Proton session | `~/.config/omega-cal-mail/session.pkl` — live JWT + mail keys, mode 600. Auto-refreshed and re-saved on every refresh. |
| processed ids | `~/.config/omega-cal-mail/state.json` |
| app-version cache | `~/.config/omega-cal-mail/appversion` |
| 2FA on re-login | drop the TOTP code in `~/.config/omega-cal-mail/2fa.txt`; a pending login picks it up within 30 s |
| single instance | `flock` on `~/.config/omega-cal-mail/calmail.lock` — a second `calmail.py run` (e.g. for debugging) is refused while the service runs |

Sessions last weeks to months, so logins (the only CAPTCHA-prone step) are
rare. When a fresh login is needed the daemon resolves Proton's current web
app version from Proton's own page — Proton rejects stale versions with
`5003 "web page out of date"`.

### Troubleshooting

- **`5003` / login fails after a Proton update** — delete
  `~/.config/omega-cal-mail/appversion` and restart the service; the daemon
  re-resolves the current version on next login.
- **CAPTCHA on login** — the daemon auto-solves with retries (5 attempts).
  If it gives up, the service backs off and retries; running
  `.venv/bin/python calmail.py login` by hand usually clears it.
- **Dead session** (you logged out elsewhere, password change) — the daemon
  re-logs in automatically from `env`. To force it, delete `session.pkl`.
- **Full reset** — delete `session.pkl` + `state.json`. Losing state can
  re-process one recent matching mail, which the skill's overlap check
  flags rather than silently duplicating.
- **Nothing happens to a new mail** — check `journalctl --user -u calmail`;
  common causes: service not running, subject not starting with the
  prefix, or the mail sent to a different account than the one in `env`.

## Testing the calendar side without mail

```bash
.venv/bin/python calmail.py process "dev wed 3pm smoke test"
```

runs only the headless-omp step for a synthetic message (put `dev` in the
text to keep it off production) and prints the reply line.

## Security notes

- **Nothing sensitive is in this repository.** Credentials live only in
  `~/.config/omega-cal-mail/` (mode 600): the Proton password in `env` and
  the live session in `session.pkl`. The session file is full-mailbox
  access — treat it accordingly (and note that deleting it locally does not
  revoke anything server-side).
- Anyone who can mail your account can submit calendar requests. The
  skill scopes the assistant to the shared calendar and treats mail text
  as data; the headless session runs with a single tool, a 10-minute
  cap, and an empty mode-700 scratch working directory (not your home
  dir), so a successful prompt injection has no files to reach. Still:
  treat mail content as untrusted input.
- The daemon never reads or forwards mail that doesn't match the prefix;
  non-matching mail is left untouched (and unread).

## Repository layout

| File | Purpose |
|---|---|
| `calmail.py` | the daemon (login, polling, matching, omp step, reply) |
| `calmail.service` | systemd user unit |
| `env.example` | config template (copy to `~/.config/omega-cal-mail/env`) |
| `pyproject.toml` | project metadata + dependencies |
