# Molecule Labs — agent skill

A skill plus a small MCP server that let an AI agent write files into a **Molecule Lab that
a human owns**.

The researcher makes their Lab in the [Labs app](https://labs.molecule.xyz) — email sign-in,
no wallet, no code. Their agent then gets its **own** identity rather than borrowing theirs:

| # | Actor | Action |
|---|-------|--------|
| 1 | Agent | Creates a wallet and reports the address |
| 2 | **Human** | Adds that address to their Lab as **Contributor** |
| 3 | Agent | Issues its own write token |
| 4 | Agent + **Human** | Agrees what is being uploaded, where, and whether it is public or private |
| 5 | Agent | Uploads — plaintext, or encrypted |
| 6 | Agent | Verifies it, and hands back a link |

**The human never hands over a private key, a token, or their session.** Revoking the agent
is one click that touches nothing else.

## The agent asks before it publishes

Publishing to a data room is a one-way door: a public file is downloadable by anyone with
the link the moment it lands, a path can never be reused, and nothing un-publishes what was
already fetched.

So **the agent never picks the visibility.** Step 4 is not advisory — the server stages every
upload, hands the plan back to be read out, and refuses to write anything until the human's
actual answer has been recorded. Placeholder approvals are rejected.

Two things the skill is loud about, because both quietly publish confidential data:

- The access level is a **label, not a lock**. Uploading plaintext and marking it
  confidential succeeds and stores your plaintext. Only encrypting encrypts.
- A private file's ciphertext is still downloadable by anyone who can query the Lab. The
  confidentiality is in the encryption, not the label.

## Public and private uploads

| | |
|---|---|
| **Public** | plaintext; anyone with the link can download it, permanently. |
| **Private** | AES-256-GCM encrypted before it leaves the machine. The key is released only to wallets the Lab's on-chain access conditions admit — by default its Contributors and its owner. |

The encryption is the same envelope the Labs app uses, so a file the agent encrypts opens in
the app. It is strong and rule-based but **not** zero-knowledge: Molecule operates the key
service, so Molecule's infrastructure can decrypt the file. The skill says so rather than
overselling it.

Note that the default is one notch tighter than the app: read-only **Viewers** cannot open a
file the agent encrypts unless you ask for `conditionRole="viewer"`.

## What it costs

Nothing, on either path. The agent's key signs exactly one off-chain message and never sends
a transaction — no gas, no tokens, no payments. Encryption adds no cost.

## What you need

| | |
|---|---|
| **A Molecule API credential** | The `mol_…` string. Think of it as an API key — it says whose calls these are and can be revoked. Starter-pack users have one; otherwise ask on the [Molecule Discord](https://t.co/L0VEiy4Bjk). |
| **Your Lab** | Just the URL from its page. Nobody needs to find its 32-byte id. |

That is the whole list. There is nothing to install by hand: no Python, no `pip`, no
virtualenv, no uv. The plugin sets itself up the first time it runs (see below).

## What is in this repository

This repository is a **marketplace** that contains **one plugin**. That is why
`.claude-plugin/` appears twice — they are two different, host-mandated files:

```
.claude-plugin/marketplace.json        the catalogue: "this repo offers these plugins"
LICENSE, README.md                     repository-level, shared
plugins/
└── molecule-lab-contributor-agent/    the plugin itself — point --plugin-dir here
    ├── .claude-plugin/plugin.json     the manifest: name, version, what it ships
    ├── .codex-plugin/plugin.json      the same, in Codex's format
    ├── .mcp.json                      the mol-labs server entry (every host reads this)
    ├── .env.example                   a template — never filled in here, see Configuration
    ├── hooks/hooks.json               the first-run setup hook
    ├── mcp/                           the server and its launchers
    └── skills/molecule-lab-contributor/SKILL.md
```

Neither `.claude-plugin` file can be renamed or merged: Claude Code, Grok and Codex all look
for exactly those names in exactly those places. The plugin sits under `plugins/` rather than
at the root because a marketplace entry pointing at the root resolves in Claude Code but
silently lists nothing in Grok.

Two directories are created at runtime and are gitignored, never committed:
`plugins/molecule-lab-contributor-agent/.plugin-data/` (the private uv, Python and `.env`)
and `mcp/__pycache__/`.

## Install

Paste these two lines into the Claude Code prompt, one at a time — in the Claude desktop
app's **Code** tab as much as in a terminal. No terminal is needed.

```
/plugin marketplace add moleculeprotocol/molecule-lab-contributor-agent
/plugin install molecule-lab-contributor-agent@molecule-lab-contributor-agent-marketplace
```

Driving it with something other than Claude — Grok, or any MCP client — is supported and
takes about as long: see [Running it on another agent](#running-it-on-another-agent).

The first session after installing sets itself up, with a status line that says so, and the
`mol-labs` tools are normally ready in that same session. Measured on a clean machine with
nothing pre-installed — uv, a Python and forty-nine packages all downloaded from scratch —
setup took **7 seconds**, well inside the 30-second window the host gives an MCP server to
start. On a slow connection it can exceed that; the tell is the line Claude Code prints when
the install finishes. `Plugin is now active.` means the tools are there. `Run /reload-plugins
to activate.` means type that once. Every session after that starts instantly.

`/plugin install` opens the plugin's details and asks for a scope. **User** installs it for
you everywhere and is the right answer unless you know otherwise; **Project** writes it into
the repository's `.claude/settings.json` for everyone; **Local** is this repository, you only.

Or from a clone:

```bash
git clone https://github.com/moleculeprotocol/molecule-lab-contributor-agent.git
claude --plugin-dir /path/to/molecule-lab-contributor-agent/plugins/molecule-lab-contributor-agent
```

The repository is a marketplace whose one plugin lives in `plugins/`, so `--plugin-dir`
points at the plugin, not at the checkout root. A `--plugin-dir` copy overrides an installed
plugin of the same name for that session, so you can test a change without uninstalling.

### Check it worked, on any host

One question, and it answers itself:

```
ask your agent: run config_doctor
```

`config_doctor` is the only tool that needs no credential and no wallet, so it is the right
first call on every host. Read three fields:

| Field | What it tells you |
|---|---|
| `secretsFile` | The one and only `.env` this server reads and writes. Every other guess about its location is wrong. |
| `configLoadedFrom` | Which file each variable actually came from — the fastest way to catch a stale `.env` or an inherited environment variable. |
| `issues` / `fixes` | What is missing, in the order to fix it. |

If the agent replies that it has no `mol-labs` tools, setup has not finished. On Claude Code
type `/reload-plugins`; on Grok and Codex, start a new session. If it still has none, read
`bootstrap.log` in the data directory `config_doctor` would have named — on Claude Code that
is `~/.claude/plugins/data/<plugin>-<marketplace>/bootstrap.log`.

### What to say first

The skill drives an eight-step flow and the agent follows it; you do not have to know the
tool names. Start it with plain words:

```
I want you to upload files into my Molecule Lab.
My credential is mol_… and my Lab is https://labs.molecule.xyz/labs/<your-lab>
```

The agent will save the credential, create its own wallet, and then **stop** and give you an
address. Add that address to your Lab as a **Contributor** in the Labs app, tell the agent it
is done, and it carries on. It will not upload anything until it has read the plan back to
you and you have answered whether the file is public or private.

### How the plugin sets itself up

Two pieces, and a beginner never sees either:

**A `SessionStart` hook** runs `mcp/bootstrap.sh` at the start of every session. On the
first run it does two things, both into the plugin's own persistent data directory
(`~/.claude/plugins/data/<plugin>-<marketplace>/`), touching nothing else on the machine:

1. Puts a private copy of [uv](https://docs.astral.sh/uv/) there — copying the one already
   on PATH if there is one, otherwise downloading it with the official installer.
2. Asks that uv to build the server's environment ahead of time: its own Python, plus the
   packages the server declares.

Every later run compares a hash of `server.py` to a stamp and does nothing unless the server
changed. On Windows the hook runs under Git Bash, which the Code tab already requires.

**A launcher** is what the server entry in `.mcp.json` actually starts: `mcp/launch` on macOS and Linux,
`mcp/launch.cmd` through `cmd.exe` on Windows — one entry, resolved per platform by
`${COMSPEC:-…}`. It exists before setup has run, which matters: Claude Code spawns the server
at the same moment the hook starts, and a spawn that fails is remembered for fifteen minutes.
The launcher waits for setup to finish (or runs it itself if the hook never fired), then hands
over to the private uv. Nothing depends on PATH, on a Python being installed, or on the
desktop app inheriting your shell environment.

The setup log is `bootstrap.log` in that data directory. If setup fails — usually because
the machine was offline — the agent is told so in words and relays it.

The server entry lives in `.mcp.json` beside the plugin manifest, and it is the only copy —
neither manifest repeats it. That file, not a `mcpServers` path in `plugin.json`, is what
every host reads: Claude Code picks up a plugin's `.mcp.json` on its own, and it is the only
form Grok reads at all. The entry needs the plugin variables a host sets only for plugins, so
working on this repository means running it as one — `claude --plugin-dir plugins/molecule-lab-contributor-agent`
from the checkout root. Run by hand, the launchers fall back to a gitignored `.plugin-data/`
folder inside the plugin directory.

The plugin sits in `plugins/` rather than at the repository root for the same
cross-host reason: a marketplace entry whose `source` is the root resolves in Claude Code but
not in Grok, which silently lists no plugin at all.

### How the Python dependencies get installed

There is no `requirements.txt`, and that is deliberate — **the dependency list lives inside
`mcp/server.py` itself**, in a [PEP 723](https://peps.python.org/pep-0723/) block at the top:

```python
# /// script
# requires-python = ">=3.10"
# dependencies = ["mcp>=1.2.0", "httpx>=0.27", "cryptography>=42", "eth-account>=0.13.7"]
# ///
```

uv reads that block, resolves the four dependencies (about 50 packages once transitive ones
are counted), fetches a Python that satisfies `requires-python`, and runs it — all into the
plugin's data directory. Nothing is compiled from source, so no build toolchain is needed on
any platform.

Budget about **220 MB** of disk for that directory — 216 MB measured on a fresh install: 35 MB
of uv, 69 MB of Python, and 112 MB of package cache and built environment. Nothing is pinned,
so the exact Python and package versions are whatever is current on the day you install; two
installs a month apart will not match. Each plugin update whose `server.py` differs
builds a second environment (~57 MB) beside the first, and nothing prunes the old one, so the
directory grows with the updates you take. Deleting it is safe once the plugin is
uninstalled — but not before, because your `.env` lives there too.

One list, in one place, that cannot drift from the code that imports it.

**If the server does not connect**, read `bootstrap.log` in the plugin's data directory
first; then ask the agent to run `config_doctor`.

## Running it on another agent

Nothing here is specific to Claude. The whole flow lives in the `mol-labs` MCP server and
its two secrets; the skill file is just the playbook the driving agent follows. Any agent
that speaks MCP over stdio and can hold an eight-step plan can run it.

### Grok

[Grok Build](https://docs.x.ai/build/overview), xAI's CLI agent, reads Claude Code
marketplaces, plugins, skills and MCP servers directly, so this installs with the same two
lines and no config authoring:

```
grok plugin marketplace add moleculeprotocol/molecule-lab-contributor-agent
grok plugin install molecule-lab-contributor-agent --trust
```

If Grok answers that it "couldn't scan every marketplace" it is refusing to guess between
sources, not failing on this one — any unreachable marketplace you have added, from Claude or
otherwise, triggers it. Pin the source and it proceeds:

```
grok plugin install molecule-lab-contributor-agent@molecule-lab-contributor-agent-marketplace --trust
```

`--trust` is not optional. Without it Grok finds the plugin but leaves its hooks and its
MCP server inactive, which looks exactly like a broken install. Plugins are also off until
enabled — press <kbd>Space</kbd> on it in the `/plugins` modal, or list it in
`~/.grok/config.toml`:

```toml
[plugins]
enabled = ["molecule-lab-contributor-agent"]
```

Grok reads the skill too, so the agent gets the same playbook and the same confirmation
gate it has under Claude Code. Confirm the install with `grok plugin details
molecule-lab-contributor-agent`, which prints the component inventory and the install path,
then check it the same way as anywhere else — ask the agent to run `config_doctor`.

`grok plugin uninstall molecule-lab-contributor-agent` removes it again; `grok plugin
marketplace remove <name>` removes the source and everything installed from it.

One thing to know about the first run. Grok gives an MCP server **30 seconds** to start,
and the first launch is the one that downloads a private Python and about fifty packages —
longer than that on a cold machine. The first session comes up with no `mol-labs` tools;
setup carries on in the background and the next session connects normally.

If you would rather not spend a session on that, register the server yourself with a longer
budget. It needs the whole entry, not just the timeout — an `[mcp_servers.*]` section
without a `command` is skipped. `grok plugin details molecule-lab-contributor-agent` prints
the install path:

```toml
[mcp_servers.mol-labs]
command = "<install path>/mcp/launch"
startup_timeout_sec = 600
```

**What will not work is Grok's hosted MCP** — the API's `mcp` tool type and the custom
connectors on grok.com. Both accept only Streamable HTTP and SSE, and they reject
`localhost` and private addresses outright. This server is stdio and holds your credential
and the agent's key on your machine, so there is nothing to point them at.

Please do not reach for a tunnel to get around that. Wrapping the server with
supergateway or ngrok does technically work, and it publishes an unauthenticated
upload-and-read tool surface — backed by the agent's signing key — on a public URL, to be
called by anyone who learns it. Use an agent that can start a local process instead.

### Codex

```
codex plugin marketplace add https://github.com/moleculeprotocol/molecule-lab-contributor-agent.git
codex plugin add molecule-lab-contributor-agent@molecule-lab-contributor-agent-marketplace
```

That installs the skill and enables the plugin. The MCP server takes one more line, and the
reason is worth stating because it looks like a bug otherwise: **Codex treats an MCP command
as a literal path.** It expands no variables — `${CLAUDE_PLUGIN_ROOT}`, `${CODEX_PLUGIN_ROOT}`
and `${COMSPEC:-…}` all reach the OS unexpanded — and it does not resolve one relative to the
plugin either. So the entry in `.mcp.json`, which is what Claude Code and Grok both read,
cannot work there. Codex registers it, lists it as enabled, and the server never starts.

`codex plugin list` prints the installed plugin root. Point Codex at the launcher inside it:

```
codex mcp add mol-labs -- /path/to/installed/plugin/mcp/launch
```

No environment block, and nothing on `PATH`: the launcher works out its own paths, exports
them, and installs its own uv and Python on first run. Codex hands an MCP server no plugin
variables at all, which is exactly why the launcher does that for itself.

One thing Codex will not do for you: its `SessionStart` hook does not fire under
`codex exec`, so the first run bootstraps inside the server's own startup instead of ahead
of it. That first start is slow; later ones are not. Give it one session to settle, start a
second, and ask the agent to run `config_doctor`.

`codex mcp list` shows the server, `codex mcp remove mol-labs` takes it out again, and
`codex plugin remove molecule-lab-contributor-agent` removes the plugin and its skill.

### Any other MCP client

opencode, Goose, Cline, Codex and VS Code agent mode all spawn local stdio servers, and most
let you choose the model behind them. Clone the repository and register the launcher inside the
plugin folder — `plugins/molecule-lab-contributor-agent/mcp/launch`, not `mcp/server.py`.
The launcher installs its own private uv and Python on first run and passes the plugin's
paths to the server, so there is nothing to put on `PATH` and no environment to set:

```json
{
  "mcpServers": {
    "mol-labs": {
      "command": "/absolute/path/to/molecule-lab-contributor-agent/plugins/molecule-lab-contributor-agent/mcp/launch"
    }
  }
}
```

Adapt the shape to whatever the client uses — a `command` and no arguments is all of it.
On Windows, run `plugins/molecule-lab-contributor-agent/mcp/launch.cmd` through `cmd.exe` instead.

Then give the agent the skill file as context —
`plugins/molecule-lab-contributor-agent/skills/molecule-lab-contributor/SKILL.md`. Claude
Code and Grok load it from the plugin; every other client needs it pasted in or referenced as a
system prompt, and without it the agent has the tools but not the rules — including the
one that matters, which is that it must never choose public or private for you.

Run this way, the plugin's data directory is `plugins/molecule-lab-contributor-agent/.plugin-data/`
— gitignored, inside the plugin folder rather than the checkout root — and that is where
`.env` and the agent's key live. Keep the checkout.

## Configuration

Two secrets, and the agent writes both for you:

```
"save my Molecule credential: mol_…"   -> writes MOLECULE_CONSUMER_CREDENTIAL to .env
"create your wallet"                   -> writes MOLECULE_AGENT_PRIVATE_KEY to .env
```

There are two cases, and you never have to work out which one you are in: **`config_doctor`
reports the exact path as `secretsFile`**, and that is the only answer worth trusting.

Installed as a plugin, `.env` lives in the host's plugin data directory — under Claude Code
that is `~/.claude/plugins/data/<plugin>-<marketplace>/.env` — which reads as
a doubled name here only because this marketplace is named after the plugin it carries — so it
survives updates and does not depend on which folder you opened. Registered by hand from a
clone, or under a host that passes no plugin
variables, the launcher points it at `plugins/molecule-lab-contributor-agent/.plugin-data/.env`
instead.

It is created at mode 0600 and is gitignored. **Do not keep a second `.env` anywhere near
the checkout.** The server does read one there — it searches the data directory, the plugin
folder, the directory the host was started in, and every parent of all three — but the first
file to define a key wins and the agent's own writes always go to `secretsFile`. A stale copy
therefore keeps supplying an old credential while your edits to the real file appear to do
nothing. To fill one in by hand, copy
`plugins/molecule-lab-contributor-agent/.env.example` to the `secretsFile` path rather than
copying it in place. Two gotchas worth knowing:

- On macOS `.env` is hidden in Finder — `Cmd+Shift+.` shows hidden files.
- The server reads configuration once at startup, so reconnect it (`/mcp`) after editing.

Everything else has a working default. The plugin's `.env.example` lists the endpoint
overrides, which exist so the Molecule team can point an agent at another deployment for
testing; you should not need to touch them.

### Where configuration comes from, in order

The server resolves every variable once, at startup, and the first source to define a key
wins:

1. **A real environment variable** in the process. Beats everything below it.
2. **`.env` files**, searched in this order: the plugin's data directory, the plugin folder,
   the server's own `mcp/` directory, the directory the host was started in, and every parent
   of all four.
3. **The `env` block of the nearest `.claude/settings.json` or `settings.local.json`**, found
   by the same walk. The first `.claude` directory that has an `env` block wins outright and
   the search stops there. `~/.claude/settings.json` is deliberately skipped — a value there
   would apply to every project at once.

Nothing in 2 or 3 ever overwrites something already set by a source above it, and no value is
logged. `config_doctor`'s `configLoadedFrom` names the exact file each variable came from,
which is the only reliable way to see what actually happened.

> **This is how a test picks up someone else's settings.** A `.env`, or a `.claude/`
> `env` block, in *any parent directory of the folder you open* is read. If a sibling project
> higher up the tree sets `MOLECULE_LABS_URL`, `MOLECULE_SERVICE_TOKEN` or `EVM_RPC_URL`, your
> session silently inherits them and talks to the wrong deployment with the wrong token.
> Before trusting a clean-environment test, run `config_doctor` and read every path in
> `configLoadedFrom`. Working from a directory outside the tree that holds those settings is
> the simplest way to be sure.

### Pointing it at another deployment

The four overrides — `MOLECULE_LABS_URL`, `MOLECULE_CLIENT_URL`, `MOLECULE_CHAIN_ID` and
`MOLECULE_ACCESS_RESOLVER` — must be set as a **matched set** describing one deployment.
Mixing them uploads private files that can never be decrypted, because the lock names a
contract that deployment cannot evaluate.

The awkward part is timing: on a first install the `.env` does not exist yet, because the
server creates it. Three ways round that, one per host.

**Claude Code** — put them in the `env` block of a `.claude/settings.json` in the folder you
open. This works on the very first run, before any `.env` exists, and it travels with the
folder rather than the machine:

```json
{
  "env": {
    "MOLECULE_LABS_URL": "https://…/graphql",
    "MOLECULE_CLIENT_URL": "https://…",
    "MOLECULE_CHAIN_ID": "…",
    "MOLECULE_ACCESS_RESOLVER": "0x…"
  }
}
```

Alternatively, let the plugin run once, ask the agent for `config_doctor`'s `secretsFile`,
write the four lines into that file, and reconnect with `/mcp`.

**Grok** — register the server yourself with the values attached, which also lets you raise
the 30-second startup budget the first run needs:

```bash
grok plugin details molecule-lab-contributor-agent    # prints the install path
grok mcp add mol-labs -e MOLECULE_LABS_URL=https://…/graphql -e MOLECULE_CHAIN_ID=… \
  -- <install path>/mcp/launch
```

**Codex** — the same, on the `codex mcp add` line you already have to run:

```bash
codex mcp add mol-labs --env MOLECULE_LABS_URL=https://…/graphql \
  --env MOLECULE_CHAIN_ID=… -- /path/to/installed/plugin/mcp/launch
```

Because these arrive as real environment variables they outrank every `.env`, which is what
you want for a deliberate test and *not* what you want permanently — a variable set this way
silently overrides the file the agent writes. Whichever route you take, confirm it landed:
`config_doctor` prints the resolved endpoint and the source of every value.

## Removing it, and starting from a clean slate

Uninstalling through the host leaves the plugin's data directory behind on purpose — it holds
your `.env`, and that file holds the agent's private key. **That key is the identity your Lab
granted the Contributor role to.** Delete it and the next run generates a different agent that
has no role, and someone has to grant it again in the Labs app. Copy the file somewhere safe
before removing anything you cannot get back.

For a genuinely clean test — nothing cached, nothing inherited — remove all six of these.
Only the first is obvious:

```bash
claude plugin uninstall molecule-lab-contributor-agent@molecule-lab-contributor-agent-marketplace
```

1. `~/.claude/plugins/data/<plugin>-<marketplace>/` — the private uv, Python, and your `.env`
2. `~/.claude/plugins/cache/<marketplace>/` and `~/.claude/plugins/marketplaces/<marketplace>/`
3. The entries in `~/.claude/plugins/known_marketplaces.json` and `installed_plugins.json`
4. `enabledPlugins` **and `extraKnownMarketplaces`** in `~/.claude/settings.json` — an entry
   left in the second one silently re-adds the marketplace on the next session
5. `~/.claude/skills/molecule-lab-contributor/`, if a copy of `SKILL.md` was ever placed there
   by hand. A personal skill loads in every session whether the plugin is installed or not,
   and a stale copy is invisible until it contradicts the plugin's own
6. Any `.env` or `.claude/` `env` block in the directory you plan to test from, **or in any
   parent of it** — see the warning above

On Grok, `grok plugin marketplace remove <name>` does 1–3 in one step. On Codex, remember
`codex mcp remove mol-labs` as well as `codex plugin remove …`, because the server was
registered separately.

## Privacy and data

The server runs on your machine and talks to three places: the Molecule Labs API, the
file storage that API hands it a signed upload or download link for, and a public read-only
node for the configured chain, used to check a private file's access rule before encrypting.
There is no telemetry, no analytics, and nothing is sent anywhere else.

What leaves your machine is what you asked to upload, plus the Lab id and the agent's
address needed to do it. A private file leaves as ciphertext; its key goes to Molecule's
key service, which releases it only to wallets the Lab's access rule admits. Your `mol_…`
credential, the agent's key, and its token stay in the local `.env` and are redacted from
every tool result before the agent sees it.

How Molecule handles the data that reaches its services is covered by the
[Molecule privacy policy](https://molecule.xyz/privacy-policy).

## Support

- Bugs and questions: [GitHub issues](https://github.com/moleculeprotocol/molecule-lab-contributor-agent/issues)
- Molecule community: the [Molecule Discord](https://t.co/L0VEiy4Bjk)
- Security concerns: email vladimir@molecule.to rather than opening a public issue.

## Scope

This is the **contributor** lane: the agent owns no Lab and creates none. To have an agent
mint and register a Lab of its own, use the
[`mol-labs-plugin`](https://github.com/moleculeprotocol/mol-labs-plugin) `aura-orchestrator`
skill instead.

## License

Apache-2.0 — see [LICENSE](LICENSE).
