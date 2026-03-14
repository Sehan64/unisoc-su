# unisoc-su

**Interactive root shell plugin for Unisoc/Spreadtrum devices**

Based on [unisoc-su by Skorpion96](https://github.com/Skorpion96/unisoc-su) — a method for CVE-2022-47339 to connect to `cmd_skt` and obtain a root shell on unpatched Unisoc models.

> SeHan | Elite  > built on [unisoc-su by Skorpion96](https://github.com/Skorpion96/unisoc-su)

---

## Overview

This is an AxManager plugin by me that builds on Skorpion96's unisoc-su to provide a full interactive root shell experience. It wraps the CVE-2022-47339 exploit chain into an installable plugin with a live-streaming shell client, background job control, and session recovery — all accessible directly from AxManager's QuickShell.

**This is not persistent root.** Rootbridge runs only for the current session and must be restarted after a reboot.

---

## Compatibility

| | |
|---|---|
| SoC | Unisoc / Spreadtrum |
| Required app | `com.sprd.engineermode` (sharedUser UID must be 1000) |
| Max security patch | **2025-06-05** |
| Framework | AxManager (with plugin + QuickShell support) |

Devices patched after June 5 2025 are not supported — the patch closes the `cmd_skt` access that the exploit relies on.

---

## How the exploit works

CVE-2022-47339 allows the EngineerMode app (`com.sprd.engineermode`), which runs as `android.uid.system` (UID 1000), to access the `cmd_skt` Unix socket. This socket is the interface for `cmd_services`, a Unisoc-specific daemon that runs as UID 0 (root).

The `cli-pie` binary bundled with the SysTool companion app connects to `cmd_skt` and executes shell commands as root. This plugin uses that binary as the escalation primitive.

---

## Installation & usage

### Step 1 — Install the plugin

Install `unisoc-su.zip` in AxManager like any other plugin.

### Step 2 — Set up the reverse shell listener (first time only)

The action script connects to EngineerMode via a local reverse shell on `127.0.0.1:1234`.

1. Run the AxManager action for unisoc-su — it will open Engineer Mode automatically
2. Inside Engineer Mode, go to **Debug & Log → Adb shell**
3. In the command field, type and run:
   ```
   nc -s 127.0.0.1 -p 1234 -L sh -l
   ```
4. Run the action again — it will detect the listener and connect

### Step 3 — Run the action

Tap the action button in AxManager. The action will:
- Connect to the EngineerMode shell via `nc`
- Enable `persist.sys.cmdservice.enable`
- Locate the `cli-pie` binary from the SysTool app
- Launch `setup.sh` through `cli-pie`, which starts the rootbridge daemon as root

A successful run looks like:
```
[+] Running as system user
[#] Enabling cmdservice...
[+] cmdservice enabled
[#] Locating cli-pie binary...
[+] Using cached path: /data/app/.../cli-pie
[#] Launching rootbridge...
[+] Rootbridge started successfully
```

### Step 4 — Open the root shell

In AxManager's **QuickShell**, type:
```
unisoc-su
```

You will get an interactive root shell:
```
root@RE5C9F:/ #
```

---

## Shell usage

```
unisoc-su                  enter interactive root shell
unisoc-su -c <command>     run a single root command non-interactively
unisoc-su -h               show help
```

**Built-in shell commands:**

| Command | Description |
|---|---|
| `q` / `quit` | Exit the shell (rootbridge keeps running) |
| `exit` | Exit the shell and shut down rootbridge |
| `kill` | Stop the currently running background command |
| `attach` | Reattach to a running background command |
| `help` | Show usage |

**Background commands:** Long-running scripts stream output live. After 5 seconds of no new output the prompt returns automatically. Type `attach` to resume watching or `kill` to stop the command.

---

## How rootbridge works

`setup.sh` runs inside the `cmd_services` root context and creates a file-based IPC channel at `/sdcard/unisoc-su/rootbridge/`. The `unisoc-su` client shell (running in QuickShell as UID 1000) writes commands to `in/command.txt`, rootbridge executes them as root and writes output to `out/result.txt`, then creates `out/done` to signal completion. The client streams `out/result.txt` in real time using a byte-offset reader.

---

## Stopping rootbridge

Type `exit` in the unisoc-su shell. This sends a graceful shutdown to the rootbridge daemon.

If the shell was killed externally (e.g. via AxManager's stop button while a script was running), rootbridge may still be alive. The next `unisoc-su` launch detects the stale job automatically and recovers without any manual steps.

---

## Security notes

- **Rootbridge is a root command execution channel.** Anything that can write to `/sdcard/unisoc-su/rootbridge/in/command.txt` can run commands as root. Type `exit` when you are done.
- Commands containing the word `eval` are blocked by rootbridge as a direct safety measure.
- The exploit **only works on unpatched devices**. Check Settings → About phone → Security patch level. If it reads later than June 2025 this plugin will not work.

---

## File layout

```
/sdcard/unisoc-su/
├── unisoc-su.sh          escalation launcher (runs in EngineerMode shell)
├── setup.sh              rootbridge daemon (runs as root via cli-pie)
└── plugin-data/
    ├── path.txt          cached cli-pie binary path
    ├── app_status.txt    SysTool detection status
    ├── systools-status.txt
    └── customize.log     install log

/system/bin/unisoc-su     interactive root shell client

/sdcard/unisoc-su/rootbridge/    (runtime only, recreated each session)
├── in/
│   ├── command.txt       client writes commands here
│   └── kill              write anything here to kill the running command
└── out/
    ├── result.txt        live command output
    ├── done              completion sentinel
    └── pid               PID of the running background job
```

---

## Credits

- **SeHan (Elite)** — plugin author; AxManager plugin, rootbridge IPC design, streaming shell client, CVE-2022-47339 discovery and disclosure
- **[Skorpion96](https://github.com/Skorpion96)** — original [unisoc-su](https://github.com/Skorpion96/unisoc-su), `cmd_skt` exploitation method and research

---

## Disclaimer

For educational and research purposes on devices you own. Use responsibly.
