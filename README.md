# ESP32-S3 Waveshare AMOLED 1.8" Setup Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that scaffolds complete ESP-IDF projects for the **Waveshare ESP32-S3-Touch-AMOLED-1.8** board.

## What it does

Run `/esp32-setup` in Claude Code to generate a fully working ESP-IDF project with:

- **Audio**: ES8311 codec with I2S recording and playback
- **Display**: SH8601 1.8" AMOLED via QSPI with LVGL
- **WiFi**: Station mode with auto-reconnect
- **HTTPS APIs**: TLS-enabled HTTP client (e.g., Groq STT/TTS)
- **Touch**: FT3168 capacitive touch

The skill encodes hard-won lessons from real hardware debugging — correct init sequences, pin mappings, driver quirks, and component versions that are verified to work together.

## Install

```bash
claude install-skill https://github.com/idiosyncraticee/esp32-setup-skill
```

## Usage

```
/esp32-setup a voice assistant that records speech and transcribes it
```

Or just `/esp32-setup` for a default voice assistant scaffold.

## Board

[Waveshare ESP32-S3-Touch-AMOLED-1.8](https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-1.8) — ESP32-S3R8 with 16MB flash, 8MB PSRAM, ES8311 audio codec, SH8601 AMOLED display, TCA9554 I/O expander, and FT3168 touch controller.

## License

MIT
