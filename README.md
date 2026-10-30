# MathBot Junior (ESP32-S3) - VSCode + PlatformIO

1. Install VSCode, then the **PlatformIO IDE** extension (it is offered automatically when the folder is opened).
2. File > Open Folder > choose this folder. Wait for PlatformIO to download the platform & libraries (the first time takes a while).
3. Buttons in the VSCode bottom bar: **Build** (checkmark), **Upload** (arrow), **Serial Monitor** (plug).
   Or from the terminal: `pio run`, `pio run -t upload`, `pio device monitor`.

The code is in `src/` (`main.ino` = the original sketch; the question-bank headers live in the same folder).
For a different board/pins: change `board =` in platformio.ini and the pin #defines in main.ino.

> Note: the on-screen texts, spoken phrases and Serial Monitor messages in the firmware are in Indonesian (the bot is designed for Indonesian-speaking children).
> Where this README quotes them literally, an English translation is given in parentheses.

## Questions read aloud (Tts.h)
1. Open `src/Tts.h` and fill in `WIFI_SSID` and `WIFI_PASS` (2.4 GHz WiFi).
2. Build & upload as usual (the ESP8266Audio library is downloaded automatically).
3. When a question appears, the question + choices A-D are read through the speaker; the timer is paused while it is being read.
   Touching the screen / pressing a button stops the reading. Other settings (volume, whether to read the choices) are at the top of Tts.h.

### Everything on screen is read aloud
The question (+ number + choices A-D), praise, "Ups! Belum tepat!" ("Oops! Not quite!"), hints, time-up, summary, menu, the Tricks & Formulas material, and all game texts are read aloud.
Answering / touching the screen / pressing a button stops the question reading immediately. Settings are at the top of `src/Tts.h`.

## Answer explanations (Explain.h) - no extra internet, no API key
When a child gets a question wrong twice (or time runs out), the screen shows a "MathBot menjelaskan" ("MathBot explains") card containing
the solution steps for that question, and the same explanation is read through the speaker.
- `src/Explain.h`  : 600 explanations (same order as the question bank). Can be edited by hand.
- `tools/make_explain.py` : generator for Explain.h (OVERWRITES Explain.h). See TUTORIAL.md.
Read TUTORIAL.md for the steps to run it and how to change the explanations.

## Answer by voice (Voice.h + Stt.h + SttConfig.h)
The child says **A / B / C / D** after switching the mic ON (question screen or the P4 button). Other sounds are ignored:
`VoiceGate` only lets through short utterances of 0.11-0.9 seconds (claps, clicks, long chatter and music are discarded), the mic stops
listening while the speaker is talking, and only A/B/C/D results are used. Choose the recognition method in `src/SttConfig.h` > `STT_MODE`:

| STT_MODE | Method | Requires |
|---|---|---|
| 1 | Online, Wit.ai | WiFi + `WIT_TOKEN` (wit.ai > app > Settings > Server Access Token) |
| 2 | **Offline**, Edge Impulse | a trained model (steps below) |
| 3 | Record training data | USB cable + `tools/rekam_suara.py` |

### Offline (STT_MODE 2): full steps
1. **Record data from the ESP32's own mic** (so it matches the device): set `STT_MODE 3`, upload, close the Serial Monitor, then
   `pip install pyserial` and `python tools/rekam_suara.py COM4 dataset 50`.
   The script asks for A, B, C, D in turn (50 times each), then `unknown` (other short words, coughs, knocking on the table; 100 times).
   Say them one at a time, repeating each letter with different pitch & distance. **Include children's voices** (at least 3-5 kids)
   because the model is only as good as the voices it is trained on.
2. Open edgeimpulse.com > New project > Data acquisition > Upload data: select all `dataset/*.wav` files,
   Label = **Infer from filename**, automatic train/test split.
3. Create impulse: Time series (window **1000 ms**, 16 kHz) > Audio (MFE) > Classification > Generate features > Train.
   The labels must be named exactly `a`, `b`, `c`, `d`, `unknown`. Check the accuracy under Model testing.
4. Deployment > **Arduino library** > Quantized (int8) > Build. Unzip the resulting folder into `lib/` of this project.
5. In `SttConfig.h`: `STT_MODE 2` and `EI_LIB_HEADER <NAME_inferencing.h>` (the file name is in `lib/<folder>/src/`).
6. Build & upload. The Serial Monitor shows `STT: ... terbaik "b" 93% -> B` ("best"). Confidence below `EI_MIN_CONF` (80%) = "Belum jelas" ("Not clear yet").

Tuning: too sensitive to noise -> raise `VG_MIN_RMS` (Voice.h); a child's voice is not picked up -> lower it.
Mode 1: the letter is correct but shows `tidak dikenali` ("not recognized") -> add the spelling that appeared to `STT_WORDS` (Voice.h).
Test the filter logic on a PC: `g++ -std=c++17 tools/test_voice.cpp -o test_voice -lm && ./test_voice`

## WiFi & reading settings (src/WifiConfig.h)
Fill in `WIFI_SSID` and `WIFI_PASS` (2.4 GHz WiFi) in `src/WifiConfig.h`, just once. That file is not changed when `Tts.h` is updated.
Reading uses: downloads that retry automatically on failure, a connection to Google that is kept alive, a short first chunk, later chunks
downloaded while the current chunk plays (`TTS_PIPELINE`), and short repeated phrases stored in flash (`TTS_CACHE`) in `Tts.h`.
Serial Monitor (115200): `TTS:` lines show WiFi, download size, free RAM, and the cause when reading is cut off;
the line `RAM bebas ... PSRAM: ...` ("free RAM ... PSRAM: ...") appears once at startup. If the audio is choppy / restarts, set `TTS_PIPELINE 0`.
Offline voice model: `lib/cebai-project-1_inferencing` (Edge Impulse, labels a, b, c, d, unknown), `STT_MODE 2` in `SttConfig.h`.

## No sound / only "Halo! Aku MathBot AI" is heard?
The most common cause: **WiFi is not connected**. Only short sentences already stored in the flash cache can be
played without internet; questions, praise, "Ups!", hints and explanations are all downloaded from Google TTS.

1. Open `src/WifiConfig.h`, fill in `WIFI_SSID` and `WIFI_PASS` (**2.4 GHz** WiFi; on a phone hotspot: turn off "5 GHz" / turn on
   "Maximize compatibility"). Upload again.
2. At startup, if WiFi fails to connect, the screen shows **"WiFi belum tersambung"** ("WiFi not connected") together with the cause
   (not filled in / name not found / wrong password / weak signal). Touch the screen to continue without sound.
3. Open the Serial Monitor (115200) and look for the following lines:

| Serial Monitor line | Meaning | What to do |
|---|---|---|
| `WiFi: !!! WIFI_SSID ... masih nilai bawaan` (still the default value) | `WifiConfig.h` has not been filled in | fill in the SSID & password, upload again |
| `WiFi: belum tersambung ... (kode 201 ...)` (not connected, code 201) | WiFi name not found | check the spelling (case matters), use 2.4 GHz, move closer to the router |
| `WiFi: belum tersambung ... (kode 202/15 ... password salah)` (wrong password) | wrong password | fix `WIFI_PASS` |
| `WiFi: TERSAMBUNG ke "...", IP ...` (CONNECTED) | WiFi is fine | continue to the `TTS:` lines below |
| `TTS: HTTP 429` / `HTTP 403` | Google is rate-limiting / rejecting requests | wait a few minutes; this service is unofficial, so it sometimes refuses |
| `TTS: HTTP -1 (connection refused)` | internet does not reach Google (school WiFi / login portal, DNS) | try a phone hotspot |
| `TTS: RAM tidak cukup` / `unduhan tidak lengkap` (not enough RAM / incomplete download) | low memory / weak signal | set `TTS_PIPELINE 0` in `Tts.h`, move closer to the router |
| `TTS: dari cache ...` (from cache) | phrase played from flash (no WiFi) | normal |

Without WiFi the device still works: questions & answers appear on screen, only the spoken reading is missing.


## Fix v4: questions & explanations not read aloud, mic not detected
**Mic**
- The mic is now **ON right away** on the question screen. Before, it was OFF even though the on-screen text said "Ucapkan: A, B, C, atau D" ("Say: A, B, C, or D"), so speech was never heard.
  The mic icon at the top-left of the question screen can still be touched to turn it off / on.
- The mic is no longer locked while the spoken audio is being **downloaded** (it used to stay locked for up to half a minute if the internet was slow / Google refused).
  Now it is only locked while sound is actually coming out of the speaker.
- At startup the device tests the INMP441 signal (Serial Monitor: `Mic: uji sinyal kanal KIRI = ...`, "signal test LEFT channel"). If it is flat, the LEFT/RIGHT channels are tried automatically in turn
  (INMP441 L/R pin to 3V3 = right, to GND = left). If both are flat: the question screen shows **"Mic tidak terdeteksi - cek kabel INMP441"** ("Mic not detected - check the INMP441 wiring").
- The question screen has a `Mic [|||  ]` indicator that rises when the child speaks, so you can see right away whether the mic hears anything.
- Serial Monitor every 3 seconds: `MIC: suara ... DIBLOKIR / mendengarkan` ("sound ... BLOCKED / listening") + the reason it is blocked.

**Questions & explanations not read aloud** (only short sentences are heard = short sentences are played from the flash cache, while questions/explanations must be downloaded)
- If a download fails, the question screen shows the cause: `Suara bacaan gagal: HTTP 429 / HTTP 403 / TLS gagal / RAM kurang / WiFi mati / internet putus` ("Reading audio failed: ... / TLS failed / low RAM / WiFi off / internet down").
- Two Google addresses are tried in turn (`translate.google.com`, then `translate.googleapis.com`).
- If 2 chunks in a row fail, downloading pauses for 15 seconds so the question & timer are not held up; retries & timeouts were shortened.
- **Without PSRAM** (Serial: `PSRAM TIDAK aktif`, "PSRAM NOT active") the device automatically uses RAM-saving mode (no parallel player, the TLS connection is released before playing the MP3).
  If your board is an ESP32-S3 **N8R8 / N16R8** (octal PSRAM), enable it in `platformio.ini`: `board_build.arduino.memory_type = qio_opi` and
  `build_flags = -DBOARD_HAS_PSRAM`. Do not enable it if your module has no PSRAM (the device will not boot).
- Pronunciation fixed: `2 : 3` in ratio/scale questions is read as "banding" (ratio), `&` = "dan" (and), `0-100` = "0 sampai 100" (0 to 100), `FPB`/`KPK` (GCF/LCM) are spelled out, `6d` = "6 detik" (6 seconds), `L` = "liter".


## Fix v5: faster sound, more sensitive mic + Mic Test
**Sound comes out faster**
- Without PSRAM: "Soal nomor N." ("Question number N.") is no longer played as a separate chunk (it used to add ~1.5 seconds before the question was downloaded); it is merged with the first sentence.
- The TLS connection to Google is kept between chunks as long as there is enough RAM for the MP3 decoder (no 1-2 second handshake per chunk); it is released automatically when RAM is tight or 6 seconds after reading ends.
- With PSRAM: chunk 1 = 30 characters, chunk 2 = 60 characters, the rest 100, so the sound starts earlier.
- The flash cache is bigger: sentences up to 100 characters (a question that was read before plays instantly next time, without internet), 900 KB total.

**Mic**
- `MIC_GAIN 3` (main.ino): software amplification. A child's speech from 20-30 cm away on the INMP441 only reaches roughly 200-300 on the scale, while the filter threshold `VG_MIN_RMS` = 220,
  so normal speech was often not counted as sound. Too sensitive -> lower it to 2; still "deaf" -> raise it to 4-6.
- **Mic Test**: on the question screen, HOLD the mic icon for 1.5 seconds. The screen shows 3 stages: (1) the mic signal bar + red threshold line,
  (2) the speech filter (passed / rejected + length in ms), (3) the recognizer result (letter A-D / not recognized / failed). The KELUAR ("EXIT") button returns to the question.


## Fix v6: Edge Impulse voice model included in the zip + next question without download delay
**What changed** (from `MathBot_ESP32S3_PIO_Lengkap`): `src/main.ino` and `src/Tts.h` were replaced with v5 + the v6 fixes below. All other files are unchanged.
The Edge Impulse library `lib/cebai-project-1_inferencing/` is already included and `SttConfig.h` is already `STT_MODE 2` + `EI_LIB_HEADER <cebai-project-1_inferencing.h>`.
Only `examples/` (for Nano 33 BLE / Portenta) and the `depends=` line in `library.properties` were removed. At build time, `main.ino` automatically checks that the model was trained at 16 kHz (same as the mic).
**Don't forget** to fill in `WIFI_SSID` / `WIFI_PASS` in `src/WifiConfig.h` (still the default values).

**The next question is heard right away (no delay)**
- Before: for every new question, the audio was first downloaded from Google (1-3 seconds, longer if the TLS connection had to be reopened).
- Now (`TTS_PREFETCH 1` in `Tts.h`): as soon as the current question has finished being read, the device quietly downloads the audio of the NEXT question into the flash cache
  while the child answers. When the next question appears, all of its chunks are already in flash -> heard instantly (works even without WiFi).
- The background download makes no sound, shows no error messages, yields instantly if there is other reading (praise, hint, etc.),
  does not disturb the mic, and backs off for 20 seconds on failure (`TTS_PREFETCH_BACKOFF_MS`). Without PSRAM and with tight RAM, the background download is skipped.
- The flash cache no longer jams when full: old phrases are deleted automatically, and recently used phrases / the next question are protected.
- Question NUMBER 1 of each session still needs one download (there is nothing that can be downloaded beforehand). A question that was read before plays instantly afterwards.
  Serial Monitor: `TTS: soal berikutnya siap di cache (N potongan baru, ... ms)` ("next question ready in cache (N new chunks, ... ms)") means the prefetch succeeded.

**Mic detected**: the INMP441 check at startup (signal, automatic LEFT/RIGHT channel), the `Mic [|||  ]` indicator, the message "Mic tidak terdeteksi - cek kabel INMP441",
and the **Mic Test** (hold the mic icon for 1.5 seconds) remain from v5 and are now used together with the Edge Impulse model above.


## Fix v7: truly delay-free reading + a more robust mic
**Question reading**
- Before the first question is shown, a **"Menyiapkan soal..."** ("Preparing the question...") screen (max 8 seconds, touch = skip) downloads that question's audio into flash. As soon as the question is shown, its audio
  plays straight from flash (no download). The download delay is moved to the loading screen, instead of sitting between the question appearing and the sound.
- Later questions are downloaded in the background while the child answers (v6). If one is not ready when moving to the next question (the child answered very quickly), the
  "Menyiapkan soal..." screen appears briefly (max 5 seconds) and then the question is read instantly. A question that has been read before does not wait at all.
- The background download pauses while the child is speaking / their speech is being processed, so RAM & CPU go to the voice recognizer.
- Without WiFi / WiFi not filled in: there is no waiting screen, the question is shown immediately (only phrases already in flash are heard).

**Mic**
- The mic now has a higher priority than the audio download (before, when WiFi/TLS was busy, pieces of the child's voice could be lost -> felt like "not heard"),
  the I2S buffer was enlarged from 8 -> 12 blocks, and the mic task stack was raised (Serial-printing `%f` could make it run out of stack).
- **Automatic re-detection**: if the INMP441 sends no signal, every 6 seconds the device retries the LEFT & RIGHT channels by itself
  (Serial: `Mic: TERDETEKSI di kanal KIRI/KANAN`, "DETECTED on LEFT/RIGHT channel"). A mic that was just plugged in / soldered is also detected without a restart.
- At startup, if the mic truly has no signal, a **"Mic tidak terdeteksi"** ("Mic not detected") screen appears with a list of wires to check (it disappears by itself once detected).
- The "has signal" threshold was lowered 2.0 -> 1.0 (`MIC_ALIVE_RMS`) because some INMP441 modules are very quiet.

**If there are still problems**, copy the Serial Monitor (115200) lines beginning with `Mic:`, `MIC:`, `STT:`, `TTS:` and `RAM bebas ... PSRAM`.


## Fix v8: smooth sound (no stutter) + the child may answer WHILE the question is still being read
**What changed** (from v7 / `MathBot_Lengkap_v11`): only `src/main.ino`, `src/Tts.h`, `src/Voice.h`. Other files are unchanged.

**1. Reading audio broken up word by word (playback, not download)**
- Main cause: in the parallel player path, phrases were written to flash (LittleFS) EXACTLY while the MP3 was playing. Writing / erasing flash sectors stalls the ESP32-S3 flash cache,
  the MP3 decoder stalls too, the speaker buffer (previously ~85 ms) runs empty, and `tx_desc_auto_clear` fills it with silence = gaps.
- Now: while reading, only short phrases (`TTS_LIVE_SAVE_MAX` = 60 characters) are saved, copied to PSRAM first, and only written to flash AFTER the whole reading is finished.
  Long questions & explanations are already filled in by the background download (prefetch).
- The speaker buffer went from 8 -> 12 blocks (`SPK_DMA_COUNT` in main.ino, ~128 ms; RAM +4 KB). The remaining buffer is now waited out (`TTS_DRAIN_MS`); before, only 60 ms, so the end of the audio was cut off.

**2. The child can answer before the question has finished being read (barge-in)**
- Before, the mic was locked during reading (+ the first 3 seconds), even in the gaps between chunks. Now there are two paths:
  - **Gap between chunks**: between reading chunks there is a `TTS_LISTEN_GAP_MS` (300 ms) silence with the mic open. Speech there immediately stops the reading and keeps being recorded.
  - **While the speaker is sounding**: the mic learns the speaker's echo peak; a child's voice that is `BARGE_RATIO` (1.8x) louder and above `BARGE_MIN_RMS` for 48 ms stops the reading.
- If it is triggered by mistake (reading is cut off but no speech passes the filter), reading resumes after 1.6 seconds and barge-in is turned off for that question (active again on the next question).
- The mic's deaf window was shortened: 100 ms after a chunk (was 300), 60 ms after reading (was 500), and a new question immediately clears the recorder.
- **Tuning**: see the Serial Monitor `BARGE: gema=.. batas=.. suara=..` ("echo=.. threshold=.. voice=.."). The child's voice must exceed the `batas` (threshold). Without echo cancellation (AEC), the second path is only reliable if the child speaks
  clearly louder / closer than the speaker. If it never interrupts: lower `BARGE_RATIO` (1.3) or reduce `TTS_VOLUME_PCT` (30). If it triggers by mistake often: raise `BARGE_RATIO` (2.2)
  or set `BARGE_IN 0` (the gap-between-chunks path is turned off too; the mic goes back to being deaf during reading).
