# unisoc-su

**Interactive root shell for Unisoc/Spreadtrum devices via CVE-2022-47339**

---

## What it is

unisoc-su is an AxManager plugin that gives you an interactive root shell on unpatched Unisoc devices. It works by connecting to the `cmd_services` socket (exposed via CVE-2022-47339) through the EngineerMode app's `cli-pie` binary, which runs as `uid=0`. A file-based IPC bridge (rootbridge) routes commands between your shell session and the root execution context, with live output streaming, background job control, and automatic session recovery.

This is not Magisk root. It does not persist across reboots by default and does not modify the system partition. It is a shell-level exploit that works while the device is unpatched.

---

## Requirements

| Requirement | Details |
|---|---|
| Device SoC | Unisoc / Spreadtrum |
| App | `com.sprd.engineermode` installed and running as `android.uid.system` (UID 1000) |
| Security patch | **2025-06-05 or earlier** |
| Framework | [AxManager](https://github.com/frb-axeron) with plugin support |
| Tools | `nc` (netcat) available on device |

---

## How it works

1. **AxManager action** (`action.sh`) checks for a reverse shell on `127.0.0.1:1234`
2. If found, it sources `unisoc-su.sh` into the EngineerMode shell via `nc`
3. `unisoc-su.sh` sets the `persist.sys.cmdservice.enable` property and launches `cli-pie` — a binary bundled with the SysTool companion app that connects to `cmd_skt` and executes as root
4. `setup.sh` runs inside the root context, creating the rootbridge IPC directory at `/sdcard/unisoc-su/rootbridge/`
5. The `unisoc-su` binary (the client shell) polls the IPC channel, forwards your commands to rootbridge, and streams output back in real time

```
AxManager action
      |
      v
EngineerMode shell (uid=1000)
      |
      | nc 127.0.0.1:1234
      v
unisoc-su.sh  -->  cli-pie  -->  cmd_skt  -->  cmd_services (uid=0)
                                                      |
                                               setup.sh (rootbridge daemon)
                                                      |
                                        /sdcard/unisoc-su/rootbridge/
                                                      ^
                                                      |
                                              unisoc-su (client shell)
```

---

## Setup

### First time

1. Install the plugin via AxManager
2. Open the AxManager action for unisoc-su
3. When prompted, Engineer Mode opens automatically — go to **Debug & Log → Adb shell**
4. In the Adb shell field, type:
   ```
   nc -s 127.0.0.1 -p 1234 -L sh -l
   ```
5. Run the action again — it will connect and launch the root shell

### Using the shell

```
unisoc-su                  enter interactive root shell
unisoc-su -c <command>     run a single root command
unisoc-su -h               show help
```

**Shell commands:**

| Command | Action |
|---|---|
| `q` / `quit` | Exit shell, rootbridge keeps running |
| `exit` | Exit shell and shut down rootbridge |
| `kill` | Stop the currently running background command |
| `attach` | Reattach to a running background command |
| `kill <pid>` | Forward kill signal to a process |
| `help` | Show usage |

Long-running scripts (e.g. monitoring loops) stream output live. After 5 seconds of no new output the prompt returns — type `attach` to reattach or `kill` to stop.

---

## Security considerations

- **rootbridge is a persistent root IPC channel.** While it is running, anything that can write to `/sdcard/unisoc-su/rootbridge/in/command.txt` can execute commands as root. Stop it with `exit` in the shell when not in use.
- **`eval` is blocked** in rootbridge as a direct command — commands containing the literal word `eval` are rejected.
- **This exploit only works on unpatched devices.** The June 2025 security patch closes the `cmd_skt` access that `cli-pie` relies on. Check your patch level in Settings → About → Security patch level.
- This tool is intended for use on your own device for research and development purposes.

---

## Stopping rootbridge

Type `exit` inside the unisoc-su shell. This sends a graceful shutdown signal to the rootbridge daemon.

If unisoc-su was killed externally (e.g. via AxManager's stop button), rootbridge may still be running. The next time you launch unisoc-su it will detect the stale job and recover automatically.

To manually confirm rootbridge is stopped:
```sh
ls /sdcard/unisoc-su/rootbridge/out/pid
```
If the file does not exist, rootbridge is not running an active job.

---

## File structure

```
/sdcard/unisoc-su/
├── unisoc-su.sh          launcher (runs inside EngineerMode shell)
├── setup.sh              rootbridge daemon (runs as root via cli-pie)
└── plugin-data/
    ├── path.txt          cached path to cli-pie binary
    ├── app_status.txt    SysTool detection status
    └── customize.log     install log
```

```
/system/bin/unisoc-su     client shell binary (interactive + -c mode)
```

```
/sdcard/unisoc-su/rootbridge/   (created at runtime, removed on restart)
├── in/
│   ├── command.txt       unisoc-su writes commands here
│   └── kill              write anything here to kill the running command
└── out/
    ├── result.txt        live command output
    ├── done              sentinel: created when command finishes
    └── pid               PID of running background job
```

---

## Known limitations

- Does not survive reboot — rootbridge must be restarted each session
- Only works on Unisoc devices with `com.sprd.engineermode` running as UID 1000
- Hard patch cutoff: **2025-06-05**. Devices patched after this date are not supported
- Output streaming uses 50ms file polling — not suitable for high-frequency real-time output
- The word `exit` anywhere in a forwarded command is blocked (prevents accidental rootbridge shutdown)

---

## Credits

- **[Skorpion96](https://github.com/Skorpion96)** — CVE-2022-47339 research, cmd_skt exploitation, core unisoc-su concept
- **SeHan** — CVE-2022-47339 original disclosure
- **Elite** — rootbridge IPC architecture, streaming shell, AxManager plugin integration

---

## Disclaimer

This tool is provided for educational and research purposes on devices you own. Use responsibly.
