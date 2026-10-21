# AGENTS.md — Study Bot / Solvunt handoff

> Read this first. It captures the whole project: the people, the hardware we
> own, every design decision we explored, why we ended up where we are, and
> what the current status is. If you are a new agent, this should be enough to
> continue without re-asking the user everything.

---

## 1. What this project is

**study-bot** is a voice "study buddy" robot: you speak a study doubt, it
understands you, asks an AI tutor (Sol, from the **study-doubt-ai** project),
and speaks the answer back, with a cute animated face.

It is being built as a **school/college hackathon project for the user's
girlfriend (GF)**. Two repos are involved and they stay separate:

- **`Ripecoder/study-bot`** (this repo, public) — the ESP32 **firmware** + the
  browser flasher page. *Currently also holds the project docs.*
- **`Ripecoder/study-doubt-ai`** (private) — the **backend** (Flask on Vercel)
  powering the site **Solvunt**, a free AI tutor. Live at
  `https://study-doubt-ai.vercel.app`.

The original plan was: ESP32 device ↔ Flask/Vercel backend ↔ Groq (speech-to-text,
LLM, text-to-speech). The device would show a face on two 16×2 LCDs.

---

## 2. TL;DR — where we stand today

- **Hardware is blocked.** The parts we own are the wrong type for the job, and
  we have **no wires, no soldering iron, no breadboard, no small components**.
  The GF has refused to buy anything more. So the physical ESP32 device **cannot
  be built right now**. See `HARDWARE-CONSTRAINTS.txt` / `.md`.
- **Firmware: not written.**
- **Backend (`study-doubt-ai`): exists and works** for text + voice (voice is
  done **in the browser** via the Web Speech API). Its system prompt was changed
  to be brief and to answer identity questions (details in §9).
- **Laptop-only path is the recommended fallback** and is essentially already
  working (the web app does mic + speech in-browser). See §10.
- **Hackathon context:** school/college hackathon. Verdict: a *polished*
  laptop-only demo can win; hardware would have more "wow" but is unbuildable,
  and a broken hardware demo is worse than a working app.
- **Unrelated but done in the same session:** Termux keep-alive scripts so the
  dev environment survives screen-off / leaving Termux (see §13).

---

## 3. People & important context

- **The user (owner):** GitHub **`Ripecoder`**. Works mostly by chatting to an
  AI agent rather than typing code. Is currently traveling; wants to build in
  bursts; phone-based dev setup (Termux → PRoot Ubuntu). Communicates tersely
  and dislikes long replies.
- **The user's girlfriend (GF):** the "human in the loop" for the school
  project. **Non-technical.** Long-distance relationship. She is on **Windows**.
  She was supposed to do the physical wiring + flashing. She does **not know
  GitHub**, prefers **WhatsApp**, and asks for **very short** messages/answers
  (we had to trim a hardware explainer down to ~2 lines for her).
- **Important:** the GF got **angry about buying more parts**, so there is a
  hard "no more purchases" constraint. Do **not** plan solutions that require
  buying stuff unless the user explicitly reopens that.
- **Honest read:** the GF **cannot build this hardware alone** (soldering,
  ~20 breadboard connections, debugging with no multimeter). Any plan that
  assumes she assembles it solo will likely end in a dead board + frustration.
- **Name / branding:** the AI tutor is **Sol** (site: **Solvunt**). Sol says
  **"Hello I am Sol."** when asked who it is, and **"I was built by Nirmal."**
  when asked who built/made/created it (Nirmal = the builder credited for Sol;
  this was an explicit user request).

---

## 4. Hardware we actually own

| Part | Notes |
|---|---|
| **ESP32 DevKit V1** (DOIT, ESP32-WROOM-32) | *Classic* ESP32, dual-core 240 MHz, **no PSRAM**. Has DACs on GPIO25/26. Auto-reset over USB-serial. |
| **8 Ω 0.5 W speaker** | Came with the ISD1820 kit. Reuse it. |
| **2× JHD162A 16×2 character LCD** | HD44780-compatible, **bare (no I2C backpack)**. Supports 8 custom 5×8 chars each. Needs 5 V, contrast pot. |
| **ISD1820 voice module** | Standalone record/play chip with onboard electret mic + amp + speaker output. **Cannot** do cloud STT/TTS. |

**Also missing (confirmed: "None of these"):** jumper wires, breadboard,
soldering iron, transistors, resistors, capacitors, 10k potentiometer,
multimeter. This is why nothing can be connected.

**Parts that were planned but are now blocked (GF won't buy):**

| Part | Qty | Why it was needed |
|---|---|---|
| INMP441 I2S MEMS mic | 1 | Digital mic the ESP32 can actually read |
| MAX98357A I2S amplifier | 1 | Drive the speaker cleanly/loudly |
| PCF8574 I2C LCD backpack | 2 | Turn each 16-pin LCD into 2 wires |
| Jumper wires | 1 set | Connect anything at all |

Approx total if ever allowed: **~$7–9**. A USB **data** cable may also be needed
for flashing (many cables are charge-only).

---

## 5. Why the hardware is blocked (the honest list)

1. **No wires** → no two modules can be connected. Absolute blocker.
2. **No soldering iron** → the LCD's 16 pins (usually unsoldered) can't be
   attached; no permanent connections possible.
3. **No microphone that works:** the ISD1820's onboard mic is analog and
   band-limited (~300 Hz–3.4 kHz). Feeding it into the ESP32 requires tapping
   its amplified output into an ADC → noisy, low quality, and **would need
   attenuation (resistors) to avoid damaging the 3.3 V pin**. Whisper would
   mishear a lot.
4. **No amplifier:** the ISD1820's amp can't be reused (it only plays its own
   recordings; no line-in). The ESP32's 8-bit DAC could drive the speaker but it
   would be **very quiet and robotic**, and really needs a transistor + resistor.
5. **LCDs need I2C backpacks or 4-bit parallel wiring** (~6 GPIOs + contrast pot
   + backlight resistor per screen). We have neither the backpacks nor a pot.
6. **Power:** two backlights + speaker + ISD1820 on the ESP32's small USB supply
   can brown out.
7. **Wake word needs decent audio;** the hacked mic makes it unreliable.

Bottom line: the parts we own are the *wrong type* for every sub-job, and the
basic connection materials are missing. Not a "clever workaround" situation.

---

## 6. Design history / decisions explored (so we don't redo them)

1. **Agent-first workflow.** The user mostly talks; the agent codes. Prefers
   big bursts rather than long interactive sessions.
2. **Browser one-click flashing** was chosen so the GF never touches a
   toolchain. Plan: GitHub Actions builds a merged `.bin` → GitHub **Pages**
   hosts an **ESP Web Tools** page + Improv Wi-Fi. GF: Chrome → URL → plug USB →
   Install → pick Wi-Fi. Requires: Pages enabled once; Chrome/Edge on desktop;
   USB driver (CH340/CP2102); a data cable. Fallback: esptool/PlatformIO.
3. **Backend split decided:** `study-doubt-ai` (Flask, Vercel) does the smart
   work; the ESP32 is a thin client. The ESP32 **never** talks to Groq directly
   (keeps the API key on the server).
4. **Cloud voice plan (not yet implemented):** add to `study-doubt-ai`:
   - `POST /api/stt` — WAV in → Groq `whisper-large-v3-turbo` → `{text}`
   - `POST /api/ask_sync` — `{question}` → Groq LLM → `{answer}` (non-streaming)
   - `POST /api/tts` — `{text}` → Groq `playai-tts` → **raw 16 kHz mono 16-bit
     PCM** (server does the transcode; ESP32 does no codec work)
   Rationale: ESP32 is weak (~300 KB RAM) → keep all audio conversion on the
   server.
5. **Speakable-text design (decided):** LLM returns **two** fields —
   `answer_markdown` (for the web UI) and `answer_spoken` (clean prose for TTS).
   `/api/tts` also runs a **defensive sanitizer** (strip `**`, `#`, backticks,
   expand `&`→and, `%`→percent, drop URLs/emojis, describe formulas) so the
   speaker never reads markup. **Not implemented yet.**
6. **Wake word "Hey Sol":** NOT possible on classic ESP32 with open tooling
   (microWakeWord needs ESP32-S3 vector instructions; custom WakeNet needs a
   paid Espressif service). **Decision: use the stock English wake word
   "Hi ESP"** (`wn9_hiesp`) via Espressif **ESP-SR / WakeNet9**, which *is*
   supported on classic ESP32. Note the model is documented as **"Hi, ESP"**
   (rhymes with "my"), not "Hey ESP".
   - Risk: the DevKit V1 has **no PSRAM** and WakeNet is memory-hungry. This is
     the #1 thing to prove on hardware. Fallback: **push-to-talk button on
     GPIO0** (the onboard BOOT button).
7. **Firmware must be ESP-IDF, not Arduino**, because ESP-SR is an ESP-IDF
   component. (`esp-skainet/examples/wake_word_detection` is the reference.)
8. **Display/face:** two JHD162A LCDs — one for **eyes**, one for an **animated
   mouth**. Use the 8 custom CGRAM characters (eye, blink, mouth frames). Needs
   I2C backpacks to be sane (blocked).
9. **ISD1820:** not usable for the assistant; optionally reuse later for simple
   sound effects (trigger via GPIO). Not important.
10. **Speaker driver choice:** MAX98357A (clean, 3 wires) was preferred; ESP32
    DAC + NPN transistor + resistor was the no-purchase fallback.
11. **Raspberry Pi alternative:** considered and **rejected** ("only ESP 32").
    (A Pi would make custom "Hey Sol" free via openWakeWord and allow offline
    Whisper/Piper, but costs ~10× and was declined.)
12. **Two-repo separation** is intentional and should be preserved.

---

## 7. Planned firmware architecture (if hardware ever unblocks)

- **Core 1 — audio:** I2S mic → WakeNet ("Hi ESP") → on wake, record until ~1 s
  silence (energy VAD) or ~5 s cap → wrap 16-bit PCM/16 kHz/mono in a WAV header.
- **Core 0 — network:** Wi-Fi + HTTPS to Vercel: `/api/stt` → `/api/ask_sync` →
  `/api/tts`.
- **Task 3 — face:** I2C LCD render task (eyes + animated mouth).
- **Flow:** wake → short "ding" + listening eyes → record → thinking eyes →
  speak PCM via I2S + animate mouth → back to idle/listening.
- **Half-duplex only:** never listen while speaking (mic would hear the speaker).
- **Latency:** ~3–6 s round trip; the "thinking" face covers it.
- **Memory:** 5 s of 16 kHz/16-bit mono ≈ **160 KB** — cap the recording or
  stream in chunks.

### Planned pin map

**INMP441 (I2S0)**
- VDD → 3V3
- GND → GND
- L/R → GND
- SCK → GPIO26
- WS → GPIO25
- SD → GPIO34

**MAX98357A (I2S1)**
- VIN → 5V
- GND → GND
- BCLK → GPIO27
- LRC → GPIO14
- DIN → GPIO13
- Speaker +/− → speaker +/−

**LCD #1 and #2 (I2C, shared bus)**
- VCC → 5V
- GND → GND
- SDA → GPIO21
- SCL → GPIO22
- (Backpacks at different addresses, e.g. 0x27 and 0x3F)

**Button (fallback / push-to-talk)**
- Leg 1 → GPIO0
- Leg 2 → GND

Power: ESP32 via USB; 5 V rail feeds amp + LCDs.

---

## 8. Backend details (`study-doubt-ai`)

- Flask app in `app.py`, deployed on **Vercel** (serverless). CORS `*`.
- **Existing endpoints:**
  - `GET /api/health` → `{status, model}`
  - `POST /api/ask` `{question}` → **SSE stream** of `{delta}` chunks, ending
    `{done:true, model}`.
- API key: `GROQ_API_KEY` env var on Vercel. Do not put keys in the repo.
- Model auto-resolution from a preference list; first available wins. Current
  top preference `llama-3.3-70b-versatile`, then `openai/gpt-oss-120b`, etc.
  `GROQ_MODEL` can force one.
- `max_completion_tokens=1024`, `temperature=0.3`.
- **Live:** `https://study-doubt-ai.vercel.app` (health at `/api/health`).
- The web app (`public/app.js`, `index.html`) already does **STT + TTS in the
  browser** (Web Speech API) — no server audio needed for the laptop version.

---

## 9. Sol's current system prompt (already deployed)

In `study-doubt-ai/app.py`, `SYSTEM_PROMPT` is active (it used to be `None`).
Changed in this session to:

- **Brief by default:** 1–2 sentence direct answers; only explain step-by-step /
  give examples when the student explicitly asks ("explain", "why", "step by
  step", "in detail", "more").
- **Identity:** "who are you / what's your name" → exactly **"Hello I am Sol."**
- **Builder:** "who built/made/created you" → exactly **"I was built by Nirmal."**
- Everything is done **in the prompt only** — an earlier hardcoded regex guard
  was deliberately **removed** (the user did not want hardcoding).
- Maths/code/ambiguity/standalone-question instructions are included.

Commits pushed to `study-doubt-ai` main; Vercel auto-deploys on push.

---

## 10. The recommended fallback: laptop-only (no hardware)

Given the hardware block, the strongest viable path for the hackathon is:

- **Option A (rejected for now):** laptop does mic/speaker/AI; ESP32 only drives
  the LCD face. *Still blocked* by LCD wiring/soldering — doesn't help.
- **Option B (recommended):** **drop the ESP32 entirely.** The laptop does mic,
  STT, LLM, TTS, speaker, and the "face" is an **on-screen animation**. Zero
  parts, zero wiring, zero soldering. The existing web app already does
  in-browser voice via the Web Speech API, so this is "open the site + add a
  face."
  - Possible next build: add an animated face (idle / listening / thinking /
    talking) to the `study-doubt-ai` web app to make it a polished demo.

---

## 11. Hackathon context

- **Type:** school / college hackathon (judging bar is achievable).
- **Honest assessment:** a polished, working laptop-only voice AI-tutor with a
  nice face and a strong story ("free tutoring for students who can't afford
  help") can win. Hardware would add "wow" but is not buildable right now, and a
  broken hardware demo is worse than a working app.
- **What wins at this level:** (1) it works live, (2) a real relatable problem,
  (3) a memorable demo, (4) polish + a confident pitch.
- **RISK TO WATCH:** the base idea ("AI tutor with voice") looks like a ChatGPT
  wrapper on its own — add a distinctive hook + polish.

---

## 12. Dev environment

- Ubuntu 26.04 in a **PRoot** container on **Android 15 (Termux)**, `aarch64`,
  root inside the container.
- Python 3.14, Node 22, git, gcc, go available. `gh` CLI authed as `Ripecoder`.
- Python venv at `/root/venvs/main`.
- apt forced to IPv4 (`/etc/apt/apt.conf.d/99force-ipv4`) to avoid hangs.
- Local clones: `/root/study-bot`, `/root/study-doubt-ai`.
- The `opencode` binary lives at `/root/.opencode/bin/opencode` (on PATH).

---

## 13. Termux keep-alive (done this session, unrelated to study-bot)

To keep the dev session alive when leaving Termux / screen off:

- `/root/stayalive.sh` — acquires wake lock, starts watchdog, opens tmux.
- `/root/stayalive-watchdog.sh` — re-asserts the wake lock forever.
- `/root/agent.sh` — runs opencode inside a persistent, wake-locked tmux session.
- `~/.bashrc` — auto-acquires wake lock + starts watchdog on every interactive
  shell.
- `/root/TERMUX-KEEPALIVE.md` — full write-up.
- User has already done: battery = Unrestricted, Termux "Acquire wakelock".
  **Still pending:** lock Termux in Recents; disable Android 15 phantom process
  killer via ADB (`settings put global settings_enable_monitor_phantom_procs
  false` + `device_config put activity_manager max_phantom_processes ...`).

---

## 14. Repos, links, local paths

- `https://github.com/Ripecoder/study-bot` — this repo (public)
- `https://github.com/Ripecoder/study-doubt-ai` — backend (private)
- `https://study-doubt-ai.vercel.app` — live backend / web app
- Local: `/root/study-bot`, `/root/study-doubt-ai`
- Docs in this repo: `AGENTS.md` (this file), `HARDWARE-CONSTRAINTS.txt`,
  `HARDWARE-CONSTRAINTS.md`

---

## 15. Open questions / risks / next steps

**Risks**
- **Groq `playai-tts` availability on the user's key is unverified.** Check
  before relying on server TTS.
- WakeNet memory fit on a **no-PSRAM** classic ESP32 (unverified).
- Hardware entirely blocked by missing wires/tools + "no more purchases".
- GF cannot assemble hardware alone.

**Next steps (pick one with the user)**
1. **Laptop-only hackathon path (recommended):** add an animated face
   (idle/listening/thinking/talking) + a clean voice loop to the
   `study-doubt-ai` web app.
2. If parts ever become available: build the ESP32 firmware (ESP-IDF + ESP-SR),
   starting with an audio I/O spike, then the wake-word spike (the de-risk
   point), then cloud round-trip, then the LCD face, then the Pages flasher.
3. Implement the backend endpoints `/api/stt`, `/api/ask_sync`, `/api/tts` +
   the speakable-text sanitizer (only needed for the device path).

**Do NOT**
- Do not re-introduce hardcoded identity replies in the backend.
- Do not merge the two repos.
- Do not assume parts can be bought.
- Do not plan a Raspberry Pi (explicitly declined).

---

## 16. Glossary

- **Sol** — the AI tutor persona. **Solvunt** — the free tutoring site.
- **study-doubt-ai** — Flask/Vercel backend + web app (Solvunt).
- **study-bot** — this repo: ESP32 firmware + flasher (original plan).
- **WakeNet / ESP-SR** — Espressif's on-device wake-word engine (ESP-IDF).
- **"Hi ESP" (`wn9_hiesp`)** — the chosen stock English wake word.
- **INMP441** — I2S digital microphone (planned). **MAX98357A** — I2S amp (planned).
- **JHD162A** — 16×2 HD44780 character LCD (we have 2). **PCF8574** — I2C
  backpack for it (planned).
- **ISD1820** — standalone analog voice recorder module (we have it; not usable
  for this assistant).
