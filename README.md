# calmail — Proton Mail → omega calendar bridge

Email your Proton account with a subject starting with `cal:` and the event
lands in the omega shared calendar — no box login, no omp, no pasting.

```
you (phone) --"cal: tues 7:20am pickup cc"--> Proton inbox
                                                 |
                    calmail (systemd user service, this box)
                                                 |
                          headless `omp -p` session + omega-cal skill
                                                 |
                          event in omega shared calendar (prod2 by default)
                                                 |
                          reply email to the sender: "✅ Pick up CC Tue ..."
```

## How it works

- A Python daemon (`calmail.py`, `protonmail-api-client`) keeps a Proton
  session alive and waits for new mail via Proton's event-polling API
  (falls back to inbox scans if the events API misbehaves).
- A message is a calendar request when its **subject** starts with the
  prefix (`cal:`, case-insensitive). The text after the prefix is the
  request; a non-empty body is appended.
- The text is fed to a headless `omp -p` session with the `omega-cal`
  skill, which does all parsing (week blocks, "exact same", am/pm
  inference, move/cancel) and talks to the omega API exactly like an
  interactive session. Prod2 is the default unless the request says `dev`.
- The final line of the session is replied to the **sender** (threaded),
  so you — or whoever sent the mail — get the ✅/❌ confirmation.
- Processed message ids are kept in `state.json`; a catch-up scan on start
  handles mail that arrived while the box was off.

## Setup (one-time, ~5 min)

```bash
cd ~/Projects/omega-cal-mail
uv sync                                            # creates .venv
mkdir -p ~/.config/omega-cal-mail
cp env.example ~/.config/omega-cal-mail/env
chmod 600 ~/.config/omega-cal-mail/env
$EDITOR ~/.config/omega-cal-mail/env                # put in your Proton creds

.venv/bin/python calmail.py login                  # first login; auto-solves
                                                   # a CAPTCHA if Proton asks
.venv/bin/python calmail.py check                  # verify the session

# install + start the service
cp calmail.service ~/.config/systemd/user/calmail.service
systemctl --user daemon-reload
systemctl --user enable --now calmail
journalctl --user -u calmail -f                     # watch it work
```

## Usage

From your phone's mail app, email any address on your Proton account:

- subject: `cal: tues 7:20am pickup cc`
- subject: `cal: week of sept 21` + the usual block in the body
- subject: `cal: move dentist to 2pm`

You'll get a threaded reply like `✅ "Pick up CC" Tue 07:20–08:20 (#57,
Assistant)`. All the usual skill conventions apply (terse one-offs, week
blocks, move/cancel, `what's on tues` — though a query reply contains the
listing).

Optional: an iOS Shortcut / Android quick-tile that pre-fills a mail to
yourself with subject `cal: ` makes it a two-tap action.

## Operations

| What | Where |
|---|---|
| logs | `journalctl --user -u calmail -f` |
| config | `~/.config/omega-cal-mail/env` (edit, then `systemctl --user restart calmail`) |
| Proton session | `~/.config/omega-cal-mail/session.pkl` — JWT + mail keys, chmod 600. Auto-refreshed; re-saved on every refresh. |
| processed ids | `~/.config/omega-cal-mail/state.json` |
| 2FA on re-login | drop the TOTP code in `~/.config/omega-cal-mail/2fa.txt`; the pending login picks it up within 30 s |

Sessions last weeks-to-months, so logins (the only CAPTCHA-prone step) are
rare. If the saved session ever dies (you logged out elsewhere, password
change), the daemon re-logs in from `env`; if Proton throws a CAPTCHA it
auto-solves, and on repeated failure the service restarts with backoff —
check `journalctl` and re-run `.venv/bin/python calmail.py login` by hand.

Delete `session.pkl` + `state.json` to fully reset (state loss just means
one possible re-process of the most recent matching mail, which the
skill's overlap check flags instead of duplicating).

## Debugging the calendar side without mail

```bash
.venv/bin/python calmail.py process "dev wed 3pm smoke test"
```

runs only the omp step for a synthetic message (creates a **dev** event —
put `dev` in the text to keep it off prod) and prints the reply line.

## Security notes

- The Proton password and the session file (live JWT + mail keys) sit in
  `~/.config/omega-cal-mail/`, mode 600 — same trust level as
  `~/.config/omega/assistant.env`.
- Anyone who can mail your account can submit requests. The skill scopes
  the assistant to the shared calendar and treats the mail text as data,
  and the headless session runs with `--tools=bash` and a 10-minute cap —
  but treat mail content as untrusted input, as always.
- The daemon never reads or forwards mail that doesn't match the prefix;
  non-matching mail is untouched (left unread).
# omega-cal-mail
