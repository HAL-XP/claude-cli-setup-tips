# Voice Mode

Push-to-talk voice input for Claude Code. Available since v2.1.69 via the `/voice` command.

## How It Works

Voice mode uses speech-to-text to convert your spoken audio into a text prompt. Hold a key, speak, release -- Claude receives it as if you typed it.

- **Activate**: `/voice` command or the `voice:pushToTalk` keybinding
- **Default keybinding**: hold spacebar to speak, release to send
- **Languages**: 20 languages supported as of March 2026
- Works in both the terminal CLI and the VS Code extension

## When Voice Mode Is Useful

- Describing complex changes while looking at code ("refactor this component to use hooks instead of classes")
- Quick follow-up corrections during active development ("no, change that function name to X")
- Brainstorming and rubber duck debugging
- When typing is inconvenient (standing, eating, reviewing on a second monitor)

## Configuration

Custom keybinding for push-to-talk:

```
voice:pushToTalk
```

Configure this in your keybindings if the default spacebar conflicts with your workflow.

## Limitations

- **Push-to-talk only** -- no always-listening mode
- **Microphone access required** -- macOS may prompt for a permission grant
- **WebSocket-based** -- may have issues behind strict corporate proxies
- **Not available everywhere** -- some SSH setups and remote environments lack audio access

## Lessons Learned

**What works:**
- Voice for high-level direction and architectural guidance
- Quick corrections mid-session ("undo that, use a map instead of forEach")
- Describing what you want while keeping your eyes on the code

**What does NOT work:**
- Dictating exact code, variable names, or file paths -- too error-prone
- Using voice in noisy environments -- recognition quality drops fast
- Expecting always-on listening -- it is push-to-talk only, by design
