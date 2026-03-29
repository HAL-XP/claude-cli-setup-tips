# TTS Voice System

A multi-engine text-to-speech chain for Claude Code sessions that generates voice responses, delivers audio via Telegram, and supports personality customization through sliders. This goes beyond Claude's built-in `/voice` mode (which handles input only) to provide full voice OUTPUT.

## The Problem

Claude Code's built-in voice mode handles speech-to-text input. You speak, Claude receives text. But there is no built-in speech output -- Claude always responds with text. For mobile-first workflows (Telegram channels, AFK monitoring), text walls on a phone screen are not practical. You need voice responses.

## Architecture

```
User speaks (Telegram voice message or push-to-talk)
  |
  v
Claude processes and generates text response
  |
  v
TTS script converts text to audio (.ogg)
  |
  +-- Engine 1: Chatterbox (local GPU, best quality)
  |   |-- Falls back if GPU unavailable or generation fails
  |
  +-- Engine 2: Edge TTS (Microsoft, free, no GPU needed)
  |   |-- Falls back if API rate-limited
  |
  +-- Engine 3: ElevenLabs (cloud API, paid, last resort)
  |
  v
Audio file sent to Telegram (or played locally)
```

The chain tries engines in order. Each engine has a timeout. If one fails, the next takes over automatically.

## The TTS Script

A single Python script that handles engine selection, text chunking, and fallback:

```python
#!/usr/bin/env python3
"""
Multi-engine TTS with automatic fallback.

Usage:
    python tts.py "Hello world" output.ogg [voice_profile] [language]

Voice profiles: butler, soft, narrator, casual, orc, auto
Languages: en (default), fr, de, es, ja, ...
"""

import subprocess
import sys
import os
from pathlib import Path

# Engine availability (detected at startup)
CHATTERBOX_AVAILABLE = False
ELEVENLABS_AVAILABLE = False

try:
    import torch
    CHATTERBOX_AVAILABLE = torch.cuda.is_available()
except ImportError:
    pass

try:
    import elevenlabs
    ELEVENLABS_AVAILABLE = bool(os.environ.get("ELEVENLABS_API_KEY"))
except ImportError:
    pass


def chunk_text(text: str, max_chars: int = 500) -> list[str]:
    """Split text into chunks at sentence boundaries."""
    sentences = text.replace(". ", ".\n").split("\n")
    chunks = []
    current = ""
    for sentence in sentences:
        if len(current) + len(sentence) > max_chars and current:
            chunks.append(current.strip())
            current = sentence
        else:
            current += " " + sentence
    if current.strip():
        chunks.append(current.strip())
    return chunks


def tts_chatterbox(text: str, output: str, voice: str = "butler") -> bool:
    """Local GPU TTS via Chatterbox (best quality, ~3-5s per chunk)."""
    if not CHATTERBOX_AVAILABLE:
        return False
    try:
        # Chatterbox uses a reference audio file for voice cloning
        ref_audio = f"voices/{voice}.wav"
        if not Path(ref_audio).exists():
            ref_audio = "voices/default.wav"

        result = subprocess.run(
            ["python", "-c", f"""
import torch
from chatterbox import ChatterboxTTS
model = ChatterboxTTS.from_pretrained(device="cuda")
wav = model.generate("{text}", audio_prompt_path="{ref_audio}")
import torchaudio
torchaudio.save("{output}", wav.cpu(), model.sr)
"""],
            capture_output=True, text=True, timeout=30
        )
        return result.returncode == 0 and Path(output).exists()
    except Exception:
        return False


def tts_edge(text: str, output: str, voice: str = "butler", lang: str = "en") -> bool:
    """Microsoft Edge TTS (free, no GPU, good quality)."""
    # Map profiles to Edge TTS voice names
    voice_map = {
        "en": {
            "butler": "en-GB-RyanNeural",
            "soft": "en-US-JennyNeural",
            "narrator": "en-US-GuyNeural",
            "casual": "en-US-AriaNeural",
            "default": "en-US-GuyNeural",
        },
        "fr": {
            "default": "fr-FR-VivienneMultilingualNeural",
            "butler": "fr-FR-HenriNeural",
        },
    }

    lang_voices = voice_map.get(lang, voice_map["en"])
    edge_voice = lang_voices.get(voice, lang_voices["default"])

    try:
        result = subprocess.run(
            ["edge-tts", "--voice", edge_voice, "--text", text, "--write-media", output],
            capture_output=True, text=True, timeout=15
        )
        return result.returncode == 0 and Path(output).exists()
    except Exception:
        return False


def tts_elevenlabs(text: str, output: str, voice: str = "butler") -> bool:
    """ElevenLabs cloud TTS (paid, highest quality, last resort)."""
    if not ELEVENLABS_AVAILABLE:
        return False
    try:
        result = subprocess.run(
            ["python", "-c", f"""
from elevenlabs import generate, save
audio = generate(text='''{text}''', voice="Antoni", model="eleven_monolingual_v1")
save(audio, "{output}")
"""],
            capture_output=True, text=True, timeout=30
        )
        return result.returncode == 0 and Path(output).exists()
    except Exception:
        return False


def generate(text: str, output: str, voice: str = "auto", lang: str = "en") -> bool:
    """Generate TTS with automatic engine fallback."""
    # Engine priority (best quality first, most reliable last)
    engines = [
        ("chatterbox", lambda: tts_chatterbox(text, output, voice)),
        ("edge-tts",   lambda: tts_edge(text, output, voice, lang)),
        ("elevenlabs", lambda: tts_elevenlabs(text, output, voice)),
    ]

    # Skip GPU engines for non-English (if they only support English)
    if lang != "en":
        engines = [e for e in engines if e[0] != "chatterbox"]

    for name, engine_fn in engines:
        print(f"Trying {name}...", file=sys.stderr)
        if engine_fn():
            print(f"Success: {name}", file=sys.stderr)
            return True
        print(f"Failed: {name}, trying next...", file=sys.stderr)

    print("All engines failed!", file=sys.stderr)
    return False


if __name__ == "__main__":
    text = sys.argv[1]
    output = sys.argv[2]
    voice = sys.argv[3] if len(sys.argv) > 3 else "auto"
    lang = sys.argv[4] if len(sys.argv) > 4 else "en"

    success = generate(text, output, voice, lang)
    sys.exit(0 if success else 1)
```

## Streaming Chunks

Long responses must be chunked and all chunks delivered. A common mistake is sending only the first chunk:

```bash
#!/usr/bin/env bash
# send-all-chunks.sh -- Generate TTS for each chunk and send to Telegram

TEXT="$1"
VOICE="${2:-butler}"
LANG="${3:-en}"

# Source credentials
source ~/.claude_credentials 2>/dev/null

# Split text into chunks (~400 chars each, at sentence boundaries)
CHUNK_NUM=0
while IFS= read -r chunk; do
    CHUNK_NUM=$((CHUNK_NUM + 1))
    OUTPUT="/tmp/tts_chunk_${CHUNK_NUM}.ogg"

    # Generate audio
    python tts.py "$chunk" "$OUTPUT" "$VOICE" "$LANG"

    if [ -f "$OUTPUT" ]; then
        # Send to Telegram
        curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendVoice" \
            -F "chat_id=${TELEGRAM_CHAT_ID}" \
            -F "voice=@${OUTPUT}" \
            > /dev/null

        rm -f "$OUTPUT"
    fi
done <<< "$(python -c "
text = '''$TEXT'''
chunks = []
current = ''
for sentence in text.replace('. ', '.\n').split('\n'):
    if len(current) + len(sentence) > 400 and current:
        chunks.append(current.strip())
        current = sentence
    else:
        current += ' ' + sentence
if current.strip():
    chunks.append(current.strip())
print('\n'.join(chunks))
")"

echo "Sent $CHUNK_NUM chunk(s)"
```

## Voice Profiles

Map profile names to TTS engine parameters:

| Profile | Character | Use Case |
|---------|-----------|----------|
| `butler` | Formal, male, British inflection | Default assistant voice |
| `soft` | Gentle, female | Calmer responses |
| `narrator` | Deep, authoritative | Status reports, recaps |
| `casual` | Conversational, upbeat | Informal chat |
| `orc` | Gruff, fantasy character | Easter egg / fun |
| `auto` | Selected by content analysis | Adapts to mood/tone |

### Auto Profile Selection

The `auto` profile analyzes the text content and selects an appropriate voice:

```python
def select_profile(text: str) -> str:
    """Auto-select voice profile based on content."""
    text_lower = text.lower()

    # Fun/easter egg detection
    if "zog" in text_lower or "warchief" in text_lower:
        return "orc"

    # Error/critical detection
    if any(w in text_lower for w in ["error", "critical", "failed", "broken"]):
        return "narrator"  # serious tone for bad news

    # Status report detection
    if any(w in text_lower for w in ["status", "report", "summary", "recap"]):
        return "narrator"

    # Default
    return "butler"
```

## Personality Sliders

Store personality parameters in a JSON file that the TTS script reads:

```json
{
  "humor": 40,
  "formality": 70,
  "verbosity": 50,
  "dramatic": 30
}
```

These sliders influence text rewriting before TTS generation:

```python
def rewrite_for_voice(text: str, personality: dict) -> str:
    """Rewrite text for spoken delivery based on personality sliders."""
    # High formality: keep technical terms, proper structure
    # Low formality: simplify, use contractions, casual phrasing

    # High verbosity: keep details
    # Low verbosity: compress to key points

    # High humor: add light commentary
    # Low humor: straight delivery

    # High dramatic: emphasize key points, add pauses
    # Low dramatic: flat, factual delivery

    # The rewriting can be done by a fast LLM call (Haiku)
    # or by rule-based transformations
    return text  # implement based on your needs
```

## Integrating with Claude Code

### CLAUDE.md Instructions

```markdown
## Voice System
- Generate TTS: `python tts.py "<text>" output.ogg <profile> <lang>`
- Send ALL chunks (not just chunk 1) when response is long
- Voice profiles: butler, soft, narrator, casual, orc, auto
- When replying to voice input, ALWAYS reply with voice (generate + send audio)
- Rewrite for spoken delivery: no file paths, no markdown, casual language
```

### Voice-In Detection Hook

Detect voice input and force voice output:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); if echo \"$INPUT\" | grep -qi \"attachment_kind.*voice\\|/voice\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"VOICE INPUT DETECTED. You MUST reply with voice (generate TTS audio and send it). Text-only reply is NOT acceptable.\\\"}}\"; fi'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## Edge TTS Setup

Edge TTS is the most reliable fallback (free, no GPU, works everywhere):

```bash
pip install edge-tts

# Test it
edge-tts --voice en-US-GuyNeural --text "Hello world" --write-media test.ogg

# List all available voices
edge-tts --list-voices
```

## Lessons Learned

**What works:**
- Engine fallback chain -- Chatterbox quality with Edge TTS reliability
- Sending all chunks -- long responses need 2-5 audio messages, not one truncated clip
- Voice-in = voice-out rule (enforced by hook) -- consistent UX on mobile
- Personality sliders -- the same information sounds different based on user preference

**What does NOT work:**
- GPU-only TTS without fallback -- when CUDA fails, you have no voice
- Sending only chunk 1 of a multi-chunk response -- user misses most of the answer
- Reading file paths and code out loud -- rewrite for spoken delivery first
- Always using the same voice -- gets monotonous over hours of use
