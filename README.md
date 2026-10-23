# TikTok LIVE CLI

Go live on TikTok from a terminal: create a room, get the RTMP ingest URL, and
push a video file with ffmpeg. No GUI, and no copying stream keys by hand.

> **Unofficial.** This project is not affiliated with TikTok. Automating TikTok
> LIVE may violate TikTok's Terms of Service and can get an account restricted
> or banned. Use an account you can afford to lose.

## What it does

- imports a session from an installed TikTok LIVE Studio profile (Windows) or
  from a cookie string
- calls the same private webcast endpoints Studio uses to create a live room
- reads the RTMP push URL out of the response and hands it to ffmpeg
- keeps the room alive with periodic pings
- reconnects automatically if the RTMP connection drops, resuming where it
  stopped instead of replaying from the beginning
- loops a clip, or stops after a set duration

## Requirements

| | |
|---|---|
| Node.js | 20 or newer (no npm packages are installed) |
| ffmpeg | on `PATH`, or set `TTLIVE_FFMPEG=/path/to/ffmpeg` |
| Account | allowed to go live; it must have used TikTok LIVE Studio at least once |

## Install

```bash
git clone https://github.com/<your-name>/tiktok-live-cli.git
cd tiktok-live-cli
node bin/ttlive.js help
```

To get a `ttlive` command, run `npm link`, or add `bin/` to your `PATH`.
On Windows, `bin\ttlive.cmd` works from `cmd.exe` and PowerShell.

## Log in

Windows, with TikTok LIVE Studio installed:

```bash
ttlive login --from-studio
```

Anywhere else, paste the cookies from a logged-in browser:

```bash
ttlive login --cookie "sessionid=...; sid_tt=...; uid_tt=..."
```

The session is saved to `~/.ttlive/session.json`. It is only sent to TikTok.

## Go live

```bash
ttlive go --title "Join me" --file clip.mp4
```

| option | |
|---|---|
| `--title <text>` | live title (default: `Join me`) |
| `--file <path>` | video file to stream (required) |
| `--duration <sec>` | stop after N seconds |
| `--bitrate <rate>` | video bitrate (default: `2500k`) |
| `--fps <n>` | frame rate (default: `30`) |
| `--no-loop` | play the file once, then end |
| `--no-reconnect` | do not restart ffmpeg after a dropped connection |
| `--url-only` | print the RTMP URL and exit (no ffmpeg) |
| `--dry-run` | show the request without creating a room |

Stop with Ctrl+C, or from another shell:

```bash
ttlive stop
```

## Other commands

| command | |
|---|---|
| `ttlive push --url rtmp://... --file clip.mp4` | push to a URL you already have |
| `ttlive doctor [--net]` | check ffmpeg, session, and API access |
| `ttlive end` | best-effort finish ping for the last room |
| `ttlive learn` | capture Studio's own Go Live request once (debugging) |
| `ttlive sniff` | dump the webcast requests Studio makes (debugging) |

## How it works

1. `room/create_info` returns the cover, region, and group-live defaults.
2. `room/create` creates the room. The response contains `stream_url.rtmp_push_url`.
3. ffmpeg publishes the file to that URL.
4. `room/ping/anchor` is sent every 5 seconds to keep the room alive.
5. The live ends when the RTMP push stops; there is no separate HTTP "end" call.

Requests are form-urlencoded with the same query parameters, headers, and
`x-ss-stub` (MD5 of the body) that Studio uses.

## Troubleshooting

| symptom | fix |
|---|---|
| `status_code 10011` | the device/install IDs do not match the session. Log in again, or set `TTLIVE_DEVICE_ID` and `TTLIVE_INSTALL_ID`. |
| `status_code 30005` | the account already has a living room; the CLI resumes it. |
| `ffmpeg not found` | install ffmpeg or set `TTLIVE_FFMPEG`. |
| `ffmpeg exited code 1` | the RTMP connection was closed. Check `~/.ttlive/stream.log`; the CLI retries unless `--no-reconnect` is set. |
| the live page says "LIVE has ended" | the room closed because the RTMP push stopped. Keep the stream running. |
| `403` on `tiktok.com/@you/live` from a script | TikTok serves a WAF challenge to non-browser clients. Open the page in a real browser. |

## Security

This repository contains no credentials. Session cookies live in
`~/.ttlive/session.json` and are only sent to `tiktok.com` / `tiktokv.com`.

For a container or server, pass the session through the environment instead:

- `TTLIVE_SESSION_B64` — base64 of `session.json`
- `TTLIVE_SESSION_JSON` — the raw JSON
- `TTLIVE_DEVICE_ID` / `TTLIVE_INSTALL_ID` — the IDs from the same Studio install

Treat the session file like a password: anyone who has it can act as your
account. Log out of TikTok to invalidate it.

## License

MIT — see [LICENSE](LICENSE).
