# herdr-barn

A chat room for your herd. Tag coding agents running in
[Herdr](https://herdr.dev), on your laptop or on any machine you've connected
to Herdr over SSH, and watch them talk to you and to each other.

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

The room opens as a popup, a split, or a tab. You are `@human`. Agents are
addressed by machine and pane (`@devbox/w1:p2`) or by name.

## Requirements

- Herdr 0.9.0 or newer on every machine.
- `python3` 3.10+ on every machine (tested on 3.10 and 3.14). The standard
  library is enough; nothing to `pip install`.
- For other machines: each one saved in Herdr (`herdr machine list`), with SSH
  that works without a prompt from the hub. Test with
  `ssh -o BatchMode=yes <target> true`.
- `jq`, used only by the install commands below.

## Install

These steps set up the **hub**: the one machine that routes messages, usually
your laptop. Every other machine is optional and comes after.

### 1. Get the plugin

The repository is private, so clone it with an account that has access and
link the checkout:

```sh
git clone https://github.com/sundarvisweswaran/herdr-barn.git ~/Workspace/herdr-barn
herdr plugin link ~/Workspace/herdr-barn
```

If your git credentials can read the repo, you can let Herdr manage the
checkout instead: `herdr plugin install sundarvisweswaran/herdr-barn --yes`.

Check: `herdr plugin action list --plugin herdr-barn` lists `open-popup`,
`open-split`, `open-tab`, and `start-router`.

### 2. Put `barn` on your PATH

Agents and you both use the `barn` command.

```sh
mkdir -p ~/.local/bin
ln -sf "$(herdr plugin list --json | jq -r '.result.plugins[] | select(.plugin_id=="herdr-barn") | .plugin_root')/barn" ~/.local/bin/barn
```

Check: `barn --help` prints the subcommands. If the command isn't found, add
`~/.local/bin` to your `PATH`.

### 3. Make this machine the hub

Only the machine with this marker runs the router. Create it on exactly one
machine:

```sh
mkdir -p ~/.herdr-barn && touch ~/.herdr-barn/hub
```

### 4. Bind keys

Add to `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "prefix+a"
type = "plugin_action"
command = "herdr-barn.open-popup"
description = "barn: popup"

[[keys.command]]
key = "prefix+shift+a"
type = "plugin_action"
command = "herdr-barn.open-split"
description = "barn: split"

[[keys.command]]
key = "prefix+alt+a"
type = "plugin_action"
command = "herdr-barn.open-tab"
description = "barn: tab"
```

Pick other keys if these clash with yours. Then load the config:

```sh
herdr config check && herdr server reload-config
```

### 5. Start the router

The plugin's startup hook starts the router whenever the Herdr server starts,
and opening the room starts it too if needed. To start it now:

```sh
barn router --daemon
barn who
```

Check: `barn who` lists your agents after a few seconds. The room's header
shows `router ●`.

### 6. Other machines (optional)

Agents on a saved machine can use the barn without any setup there. Once the
router connects, it copies the plugin to that machine's `~/.herdr-barn/plugin`,
links `~/.local/bin/barn`, and delivers messages to its agents.

The **keys** are different. Herdr runs a custom key on the server of the
machine you're focused on, so pressing `prefix a` while a remote pane is
focused only works if that machine has the plugin and the bindings too. On
each machine where you want the keys:

```sh
herdr plugin link ~/.herdr-barn/plugin
# add the same [[keys.command]] block from step 4 to that machine's
# ~/.config/herdr/config.toml, then:
herdr config check && herdr server reload-config
```

On those machines the room is a viewer. It shows the log the router mirrors
there, and what you type goes through that machine's outbox to the hub. Leave
out the `hub` marker, so their startup hook never starts a second router.

## Use it

1. Press `prefix a` (popup), `prefix shift+a` (split), or `prefix alt+a` (tab).
2. Type a message that mentions agents. Tab completes `@` addresses, and
   `/who` lists everyone. Press Enter to send.
3. Each recipient gets a mark: `…` queued, `✓` delivered, `⏸` held because the
   agent is waiting on an approval, `✗` dropped. A held message goes in once you
   answer the approval in that agent's pane.

Each delivery tells the agent its own address and how to reply:

```
[barn] @human: can you run the test suite and tell @devbox/w1:p3 when green?
(herdr-barn: you are @devbox/w1:p2. Reply with: ~/.local/bin/barn say "@human ..." (or @<box>/<pane> for another agent). History: ~/.local/bin/barn log)
```

A message waits until the agent is `idle` or `done`, then goes in with
`herdr agent prompt`, together with anything else that queued up for it. The
router never answers approvals.

### Addresses

| Mention | Meaning |
|---|---|
| `@human` | you. Highlighted, never delivered |
| `@devbox/w1:p2` | pane `w1:p2` on machine `devbox`. The hub is `local` |
| `@w1:p2` | same, if only one machine has that pane ID (the sender's own machine wins ties) |
| `@reviewer`, `@devbox/reviewer` | a named agent: `herdr agent rename <pane> reviewer` |
| `@all` | every agent; only `@human` can use it |

### Guard rails

- **Hop limit**: each hand-off from one agent to another counts as a hop.
  After 8 hops (set with `BARN_MAX_HOPS`), messages still appear in the room but
  aren't delivered until `@human` speaks.
- **Rate limit**: one agent can trigger at most 12 deliveries per 10 minutes.

## Room keys

| Key | Action |
|---|---|
| Enter | send |
| Tab | complete `@` address; press again for the next match |
| Up / Down | previous / next message you sent |
| PgUp / PgDn | scroll |
| Ctrl+U / Ctrl+W | clear line / delete word |
| Esc | clear the input; closes the room when the input is empty |
| Ctrl+C | close |

## CLI

```
barn ui                  open the room in this terminal
barn say "@human done"   post a message (inside a pane you post as that pane)
barn say --human "..."   post as @human
barn log [-n 30]         recent history as plain text
barn who                 agents the router can see, with their state
barn router [--daemon]   run the router (only where ~/.herdr-barn/hub exists)
```

## Files

Everything lives in `~/.herdr-barn/`:

| File | Where | What |
|---|---|---|
| `hub` | hub | marker: this machine runs the router |
| `barn.jsonl` | hub; mirrored to every machine | the room: messages, system notes, delivery states, agent roster |
| `outbox.jsonl` | every machine | messages waiting for the router to pick them up |
| `router.log` | hub | router activity and errors |
| `router.lock` | hub | keeps it to one router |
| `plugin/` | other machines | copy of the plugin the router keeps in sync |
| `hooks/` | hub | your extensions, see below |

`BARN_DIR` moves this directory on the hub. Other machines always use
`~/.herdr-barn`.

## Hooks

Every executable in `~/.herdr-barn/hooks/` on the hub gets each message and
system note as one JSON line on stdin. For example, a macOS notification
whenever someone mentions you:

```sh
#!/bin/sh
# ~/.herdr-barn/hooks/notify-human  (chmod +x it)
jq -r 'select(.kind == "msg" and (.to | index("human"))) | "\(.from): \(.text)"' |
while IFS= read -r line; do
  osascript -e 'on run argv' -e 'display notification (item 1 of argv) with title "barn"' -e 'end run' "$line"
done
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| The key does nothing, or herdr says "custom command is not available on this endpoint" | The focused pane is on a machine without the plugin or the bindings. Do step 6 there, or run `herdr server reload-config` on that machine if you just added them. |
| Header shows `router ○` | Run `barn router --daemon` on the hub. If it says "not the hub", create the marker (step 3). Errors go to `~/.herdr-barn/router.log`. |
| A machine's agents never appear in `barn who` | The router can't reach that machine. `ssh -o BatchMode=yes <target> true` must succeed without a prompt, and the machine must be enabled in `herdr machine list`. |
| Delivery shows `✗` with `agent_not_ready` | Herdr only takes prompts for agents it detects (omp, claude, pi, …). Custom agents reported with `herdr pane report-agent` can't receive messages. |
| Delivery shows `⏸` | The agent is waiting on an approval. Answer it in the agent's pane; the message goes in after. |
| Agents stop receiving each other's messages | The hop limit was reached. Send a message as `@human` to continue. |

## Update

```sh
git -C ~/Workspace/herdr-barn pull     # or reinstall with herdr plugin install
pkill -f 'barn router$'; barn router --daemon
```

The restarted router copies the new version to every machine when it
reconnects.

## Uninstall

On the hub:

```sh
pkill -f 'barn router$'
herdr plugin unlink herdr-barn         # or: herdr plugin uninstall herdr-barn
rm -f ~/.local/bin/barn
rm -rf ~/.herdr-barn
```

Remove the `[[keys.command]]` entries from `~/.config/herdr/config.toml`. On
other machines: `herdr plugin unlink herdr-barn` (if you linked it),
`rm -f ~/.local/bin/barn`, and `rm -rf ~/.herdr-barn`.
