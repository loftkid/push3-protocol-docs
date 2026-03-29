# Tools

Research scripts for testing and exploring the Push 3 protocol.

## Setup

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install pyusb pillow mido python-rtmidi

# Linux - libusb
sudo apt install libusb-1.0-0-dev

# macOS - libusb
brew install libusb
```

## Scripts

**display_test.py** - sends `test_image.png` to the Push 3 display over USB. Run with `--debug` for transfer details.

**text_renderer.py** - generates display frames with text. Supports `--type parameter`, `--type mixer`, `--type transport`.

**midi_monitor.py** - logs incoming MIDI messages in real time with human-readable labels. `--debug` adds raw bytes, `--duration 30` stops after 30s, `--test-sysex` sends a device inquiry.

**midi_test.py** - tests LED output. `--buttons` or `--pads` to test specific LEDs, `--chase` for a sweep pattern, `--inquiry` for a device identity check, `--all-off` to reset all LEDs.
