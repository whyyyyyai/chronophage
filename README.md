# Chronophage

**The no-AI watchdog for [comms-deck](https://github.com/whyyyyyai/comms-deck).**

The owner's agents (Claude, Grok, Gravity, Alice, Lumo) talk through
comms-deck. Agents stop unpredictably: they run out of usage, crash, or
end a turn without saying so. Chronophage is a plain program that keeps
an eye on them every minute and delivers what they can't pick up
themselves. No model is needed for it to run, and it keeps going while
every agent is offline.

## What it does, every minute

1. **Services.** `comms-deck.service` and `push-courier.service` must be
   running. A dead one is restarted; one that won't restart is reported.
2. **Agents.** It reads `comms-deck status --json` and, for each seat:

   | It sees | It does |
   | --- | --- |
   | busy, nothing has changed for 10 min, notes waiting (the turn ended without a Stop) | `comms-deck wake <seat> --stale`: delivers them |
   | ready, but notes have waited 5 min | `comms-deck wake <seat>` |
   | out of usage (idle until its reset) | nothing: the notes wait for the reset |
   | paused by the loop guard, a draft holding notes back, an app whose sends keep failing, an agent that isn't running, an app that hasn't checked in 30 min | tells the owner |
   | codex (paused by the owner), hidden seats | nothing |

3. **Reports.** Each problem goes to the owner's chat screen once, then at
   most once an hour while it lasts. When it clears, one quiet note says so.
4. **Status.** `~/.local/state/chronophage/status.json` holds what it saw
   and did on the last pass; `journalctl --user -u chronophage` holds the history.

## What it never does

- Run a model, or need one.
- Delete, edit or reroute a note.
- Type into a window itself. Waking goes through `comms-deck wake`, which
  keeps all of comms-deck's checks: no typing over a draft, no typing into a
  focused window, and the loop guard. A wake pastes the notes and presses
  Enter, so `--stale` waits until a seat has shown no change for 10 minutes.
- Run before comms-deck has `status --json`: leave the timer off until then,
  or every pass reports that the status command failed.
- Wake an agent that is out of usage, or one the owner has paused.

## Install

Needs comms-deck with `status --json` and `wake` (comms-deck PR #38).

```bash
gh repo clone whyyyyyai/chronophage ~/chronophage
~/chronophage/chronophage --self-test          # the decisions, against made-up seats
~/chronophage/chronophage --dry-run            # what it would do right now
ln -s ~/chronophage/systemd/chronophage.service ~/chronophage/systemd/chronophage.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now chronophage.timer
```

Check it: `systemctl --user list-timers chronophage.timer` and
`journalctl --user -u chronophage -n 20`.

## Settings

The thresholds are at the top of [`chronophage`](chronophage): 10 min for a
stale busy, 5 min for a stuck ready seat, 30 min before reporting an agent
that isn't running, 15 min for a draft, 1 hour between reminders.
`CHRONOPHAGE_COMMS_DECK` points at a different comms-deck, and
`CHRONOPHAGE_STATE` at a different state folder.

## Rules

The comms-deck rules apply: no one's name (say "the owner"), no secrets,
and changes go through a pull request reviewed by another seat.

Written by: Claude (2026-11-24), at the owner's request.
Reviewed by: Grok (651d484, 2026-11-24), who also found the bug that commit fixes:
a report only counts as told once the send worked.
