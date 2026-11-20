# ccsweep

See and stop the dev servers, databases and scripts that Claude Code chats leave running on your Mac.

To do its work, Claude Code starts dev servers, databases, test runners and headless browsers. When
you close the chat, they keep running, sometimes for days, using CPU, memory and battery. ccsweep
lists them, stops what a chat started when that chat ends, and every few minutes sweeps up anything
that slipped through.

## What it looks like

```
$ ccsweep

○ Closed chat · ~/code/shop — left running
    # PID    AGE        CPU     MEM PORTS       FOLDER                       WHAT
    1 41210  2d 4h     0.0%   84 MB 5433        ‹scratchpad›/pgdata          postgres -D ‹scratchpad›/pgdata -p 5433
    2 41388  2d 4h     0.1%   33 MB 8000        ~/code/shop/api              uvicorn app.main:app --port 8000
    3 41502  2d 4h     0.0%   16 MB 3000        ~/code/shop/web              next dev --port 3000

○ Closed chat · ~/code/blog — left running
    # PID    AGE        CPU     MEM PORTS       FOLDER                       WHAT
    4 77120  9h 12m   88.1%   93 MB -           ~/code/blog                  npm exec tsx scripts/crawl.ts

● Open chat (this one) · ~/code/api-gateway
    # PID    AGE        CPU     MEM PORTS       FOLDER                       WHAT
    5 90233  12m       0.4%   61 MB 8787        ~/code/api-gateway           npm run dev

● Open chat · ~/code/infra
    # PID    AGE        CPU     MEM PORTS       FOLDER                       WHAT
    6 41445  22h 8m    0.7%  418 MB -           -                            Docker (app)

4 left running by closed chats · type c to stop them all
Apps are never stopped automatically; pick one to have it quit.
Docker: no containers running

Stop which? numbers (1 3) · c = closed chats · a = all except apps · Enter = nothing: c
stopped   postgres -D ‹scratchpad›/pgdata -p 5433  [pid 41210, port 5433]
stopped   uvicorn app.main:app --port 8000  [pid 41388, port 8000]
stopped   next dev --port 3000  [pid 41502, port 3000]
stopped   npm exec tsx scripts/crawl.ts  [pid 77120]
```

Each row is one service, with its child processes folded in. `‹scratchpad›` is Claude Code's
temporary folder for that chat.

## Install

You need macOS, Node.js 18 or later, and Claude Code.

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/Prathameshppawar/ccsweep/main/ccsweep -o ~/.local/bin/ccsweep
chmod +x ~/.local/bin/ccsweep
~/.local/bin/ccsweep install
```

Or with npm: `npm install -g github:Prathameshppawar/ccsweep && ccsweep install`

`ccsweep install` sets up three things:

| | What it does | Where |
|---|---|---|
| Stop on close | When a chat ends, everything it started is stopped. `/clear` doesn't count; the window stays open. | A `SessionEnd` hook in `~/.claude/settings.json`. Your other settings are kept, and the old file is saved as `settings.json.ccsweep-backup`. |
| Safety net | Every 5 minutes it stops what chats left behind if they closed without running the hook, for example when the editor was force-quit. | A launchd agent, `~/Library/LaunchAgents/dev.ccsweep.sweep.plist`. Change the interval with `ccsweep install --every 10`. |
| `/ccsweep` | Ask in any chat "what has Claude left running?" or "clean up", or type `/ccsweep`. | A skill in `~/.claude/skills/ccsweep`. |

Restart Claude Code chats that are already open so the hook applies to them. macOS may tell you
once that software from your Node.js publisher, such as "Node.js Foundation", can run in the
background. That's the safety net; keep it allowed.

## Use

| Command | What it does |
|---|---|
| `ccsweep` | Lists everything chats started, then asks what to stop |
| `ccsweep list [--json]` | Lists only |
| `ccsweep clean` | Stops everything closed chats left running |
| `ccsweep stop PID...` | Stops those services, with their child processes |
| `ccsweep log` | Shows what was stopped, when, and why |
| `ccsweep status` | Shows what is installed |
| `ccsweep uninstall` | Removes the hook, safety net and skill |
| `--dry-run` | With `clean` or `stop`: shows what would stop, stops nothing |

## How it works

- **Finding them.** Every command Claude Code runs carries `CLAUDECODE=1`, `CLAUDE_PID` and
  `CLAUDE_CODE_SESSION_ID` in its environment, and so does everything that command starts. `ps -E`
  shows those markers, so ccsweep knows which processes a chat started and which chat it was.
- **Open or closed.** A chat is open while its `claude` process is running. macOS hides the
  environment of its own programs (`zsh`, `sleep` and others), so Claude's shells are also
  recognised by the shell snapshot they load. Claude Code deletes that snapshot when the chat
  closes.
- **Stopping.** Services get `SIGTERM`, then `SIGKILL` if they are still running after 5
  seconds. Postgres gets `SIGINT`, its clean "fast shutdown". Apps are asked to quit through macOS.
  Every stop is written to `~/.claude/ccsweep.log`.

## What it never touches

- Your own processes: anything without Claude's markers.
- Claude Code itself.
- Apps such as Docker Desktop or a browser, unless you pick them. Quitting Docker Desktop would
  take down containers that other chats are using.
- Docker containers and `brew services`. They run outside the chat, so ccsweep can't tell who
  started them; it only says how many containers are running.

## Limits

- macOS only, for now.
- If Claude starts a bare macOS program (for example `/bin/sleep` or `caffeinate`) and it leaves
  its shell, macOS hides its markers, so ccsweep can't trace it. Shells, and anything under a
  traced process, are found.
- Closing a chat stops what it started. To keep a server running after the chat closes, start it
  yourself in a terminal, or with `brew services` or Docker.

## Uninstall

```sh
ccsweep uninstall
rm ~/.local/bin/ccsweep
```

## License

MIT. ccsweep is an independent tool and is not affiliated with Anthropic.
