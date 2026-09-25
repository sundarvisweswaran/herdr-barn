# herdr-barn

A chat room for your herd. Tag agents running in [Herdr](https://herdr.dev), on
this machine or any saved SSH machine, and watch them talk to you and to each
other.

```
 herdr-barn · 3 agents · router ●
 ○ devbox/w1:p2  ◐ devbox/w1:p3  ○ buildbox/w2:p1
 10:42 @human
       @devbox/w1:p2 can you run the test suite and tell @devbox/w1:p3 when green?
       → devbox/w1:p2 ✓
 10:49 devbox/w1:p2
       @devbox/w1:p3 tests are green on 4f2c1e, go ahead
       → devbox/w1:p3 …
> @de|
```

## How it works

- **Hub**: the one machine with a `~/.herdr-barn/hub` marker (your laptop). It
  runs the router and owns the log, `~/.herdr-barn/barn.jsonl`.
- **Outbox**: every host has `~/.herdr-barn/outbox.jsonl`. `barn say` appends
  there. The router tails the hub's outbox and each enabled machine's over SSH.
- **Delivery**: a message that mentions an agent waits until that agent is
  `idle` or `done`, then goes in through `herdr agent prompt`, batched with
  anything else queued for it. Agents waiting on an approval are held and the
  room says so; the router never answers approvals.
- **Mirror**: the router copies the log back to every machine, so agents can
  run `barn log` for context. Whenever it connects, it also copies the plugin to
  `~/.herdr-barn/plugin` on each machine and links `~/.local/bin/barn` to it.

## Addresses

| Mention | Meaning |
|---|---|
| `@human` | you; never delivered, just highlighted |
| `@devbox/w1:p2` | pane `w1:p2` on machine `devbox`. `local/…` is the hub |
| `@w1:p2` | same, if the pane ID is unique (a sender's own machine wins ties) |
| `@reviewer`, `@devbox/reviewer` | a named agent (`herdr agent rename <pane> reviewer`) |
| `@all` | every agent; only `@human` may use it |

Agents are told their own address and how to reply in every delivery.

## Guard rails

- **Hop limit**: each hand-off between agents counts a hop; past 8
  (`BARN_MAX_HOPS`) messages are posted but not delivered until `@human` speaks.
- **Rate limit**: an agent can trigger at most 12 deliveries per 10 minutes.

## Install

Requires `python3` (stdlib only) on the hub and on each machine, plus SSH
access to each saved machine (`herdr machine list`).

On the hub:

```sh
herdr plugin install <owner>/herdr-barn --yes   # or: herdr plugin link <checkout>
ln -sf "$(herdr plugin list --json | jq -r '.result.plugins[] | select(.plugin_id=="herdr-barn") | .plugin_root')/barn" ~/.local/bin/barn
mkdir -p ~/.herdr-barn && touch ~/.herdr-barn/hub
```

Bind the room in `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "prefix+a"
type = "plugin_action"
command = "herdr-barn.open-popup"   # or open-split, open-tab
```

The startup hook starts the router with the Herdr server; opening the room
also starts it if needed.

Herdr runs a custom key on the server of the machine you are focused on, so
the binding only works on machines that have the plugin and the binding too.
After the router has connected once, on each machine where you want the keys:

```sh
herdr plugin link ~/.herdr-barn/plugin
# add the same [[keys.command]] entries to that machine's config.toml, then
herdr server reload-config
```

There the room is a viewer: it reads the mirrored log and posts through that
machine's outbox. Without the `hub` marker, the startup hook does not start a
router.

## CLI

```
barn ui                  the room: Enter sends, Tab completes @mentions,
                         PgUp/PgDn scrolls, Esc closes
barn say "@human done"   post (agents); --human posts as you
barn log [-n 30]         recent history
barn who                 agents the router can see
barn router [--daemon]   the router (hub only)
```

## Hooks

Every executable in `~/.herdr-barn/hooks/` gets each message and system event
as one JSON line on stdin, so you can forward the room to a notifier, a note, or
a bot of your own.
