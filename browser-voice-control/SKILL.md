---
name: browser-voice-control
description: >
  Build browser-based voice control interfaces using the native Web Speech API
  (SpeechRecognition). Use this skill whenever a user wants to control browser
  tabs, scroll pages, trigger UI actions, or run searches using their voice —
  entirely client-side, no external APIs or keys required. Trigger this skill
  for any request involving: voice commands in the browser, speech-to-action
  workflows, hands-free browsing, accessibility voice control, DOM manipulation
  via microphone input, or "talk to the page" interactions. Even if the user
  just says "add voice control" or "make it voice-activated", use this skill.
  All recognition runs locally via the Web Speech API — no server needed.
compatibility:
  required_browser: Chrome 33+ / Edge 79+ / Safari 14.1+ (Firefox has no support)
  api: Web Speech API — SpeechRecognition (window.SpeechRecognition || window.webkitSpeechRecognition)
  local_processing: true
  network: not required (recognition may route through OS/browser voice engine)
---

# Browser Voice Control Skill

Build hands-free, voice-driven browser experiences using the **Web Speech API** — fully
client-side, zero dependencies, zero API keys. Claude generates self-contained HTML/JS
artifacts that listen for voice commands and translate them into DOM actions.

---

## Quick Reference: Core Intent Categories

| User says…                          | Intent category       | Action                            |
|-------------------------------------|-----------------------|-----------------------------------|
| "Open a new tab"                    | `tab_control`         | `window.open()`                   |
| "Close this tab"                    | `tab_control`         | `window.close()`                  |
| "Scroll down / up"                  | `scroll`              | `window.scrollBy()`               |
| "Go to top / bottom"                | `scroll`              | `window.scrollTo()`               |
| "Search for [query]"                | `search`              | focus search input + submit       |
| "Click [button/link name]"          | `dom_click`           | fuzzy-match element + `.click()`  |
| "Fill [field] with [value]"         | `dom_fill`            | target input + `.value =`         |
| "Go back / forward"                 | `navigation`          | `history.back()` / `history.go()` |
| "Refresh the page"                  | `navigation`          | `location.reload()`               |
| "Read the page"                     | `tts`                 | Web Speech `SpeechSynthesis`      |
| "Stop listening"                    | `control`             | `recognition.stop()`              |

---

## Implementation Pattern

Every voice control artifact follows this structure:

```
1. Feature-detect SpeechRecognition
2. Instantiate + configure recognition
3. Build a command registry (intent → handler)
4. Parse transcript against registry
5. Execute matched handler + give visual/audio feedback
6. Handle errors + browser compatibility warnings
```

---

## Boilerplate: Minimal Voice Control Artifact

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Voice Control</title>
  <style>
    /* --- status UI --- */
    #vc-badge {
      position: fixed; bottom: 1rem; right: 1rem;
      padding: .5rem 1rem; border-radius: 999px;
      font-family: sans-serif; font-size: .85rem;
      background: #1e1e2e; color: #cdd6f4;
      box-shadow: 0 2px 12px #0004; z-index: 9999;
      transition: background .2s;
    }
    #vc-badge.listening { background: #a6e3a1; color: #1e1e2e; }
    #vc-badge.error     { background: #f38ba8; color: #1e1e2e; }
    #vc-transcript { font-style: italic; opacity: .7; }
  </style>
</head>
<body>

  <!-- Status badge — always visible -->
  <div id="vc-badge">
    🎙 <span id="vc-status">Click to start</span>
    <span id="vc-transcript"></span>
  </div>

  <script>
  // ── 1. Feature detection ──────────────────────────────────────────────────
  const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SR) {
    document.getElementById('vc-status').textContent =
      '⚠ Web Speech API not supported. Use Chrome or Edge.';
    document.getElementById('vc-badge').classList.add('error');
    throw new Error('SpeechRecognition unavailable');
  }

  // ── 2. Instantiate ────────────────────────────────────────────────────────
  const recognition = new SR();
  recognition.continuous     = true;   // keep listening after each result
  recognition.interimResults = true;   // show partial transcripts
  recognition.lang           = 'en-US';
  recognition.maxAlternatives = 1;

  // ── 3. Command registry ───────────────────────────────────────────────────
  // Each entry: { pattern: RegExp, handler: (match) => void }
  const commands = [
    {
      pattern: /\b(open|new)\s+(tab|window)\b/i,
      handler: () => window.open('about:blank', '_blank')
    },
    {
      pattern: /\bscroll\s+down\b/i,
      handler: () => window.scrollBy({ top: 300, behavior: 'smooth' })
    },
    {
      pattern: /\bscroll\s+up\b/i,
      handler: () => window.scrollBy({ top: -300, behavior: 'smooth' })
    },
    {
      pattern: /\b(go to|scroll to)\s+(top|beginning)\b/i,
      handler: () => window.scrollTo({ top: 0, behavior: 'smooth' })
    },
    {
      pattern: /\b(go to|scroll to)\s+bottom\b/i,
      handler: () => window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' })
    },
    {
      pattern: /\bsearch\s+(for\s+)?(.+)/i,
      handler: (m) => {
        const query = m[2].trim();
        const input = document.querySelector('input[type=search], input[type=text], input[name=q]');
        if (input) { input.value = query; input.form?.submit(); }
        else window.open(`https://www.google.com/search?q=${encodeURIComponent(query)}`, '_blank');
      }
    },
    {
      pattern: /\b(go\s+)?back\b/i,
      handler: () => history.back()
    },
    {
      pattern: /\b(go\s+)?forward\b/i,
      handler: () => history.forward()
    },
    {
      pattern: /\brefresh\b|\breload\b/i,
      handler: () => location.reload()
    },
    {
      pattern: /\bstop\s+listening\b/i,
      handler: () => recognition.stop()
    },
  ];

  // ── 4. Parser ─────────────────────────────────────────────────────────────
  function parseCommand(transcript) {
    const t = transcript.trim().toLowerCase();
    for (const cmd of commands) {
      const match = t.match(cmd.pattern);
      if (match) { cmd.handler(match); return true; }
    }
    return false; // no match
  }

  // ── 5. Event handlers ─────────────────────────────────────────────────────
  const statusEl     = document.getElementById('vc-status');
  const transcriptEl = document.getElementById('vc-transcript');
  const badge        = document.getElementById('vc-badge');

  recognition.onstart = () => {
    badge.classList.add('listening');
    statusEl.textContent = '● Listening…';
  };

  recognition.onresult = (event) => {
    let interim = '', final = '';
    for (let i = event.resultIndex; i < event.results.length; i++) {
      const t = event.results[i][0].transcript;
      event.results[i].isFinal ? (final += t) : (interim += t);
    }
    transcriptEl.textContent = ` "${interim || final}"`;
    if (final) parseCommand(final);
  };

  recognition.onerror = (e) => {
    badge.classList.remove('listening');
    badge.classList.add('error');
    statusEl.textContent = `Error: ${e.error}`;
    setTimeout(() => badge.classList.remove('error'), 3000);
  };

  recognition.onend = () => {
    badge.classList.remove('listening');
    statusEl.textContent = 'Stopped — click to restart';
  };

  // ── 6. Start on user gesture (required by browsers) ───────────────────────
  document.getElementById('vc-badge').addEventListener('click', () => {
    recognition.start();
  });
  </script>
</body>
</html>
```

---

## Key Configuration Options

```js
recognition.continuous     = true;    // false = single-shot (stops after first result)
recognition.interimResults = true;    // false = only fire on final results (saves CPU)
recognition.lang           = 'en-US'; // BCP 47 tag — change for other locales
recognition.maxAlternatives = 3;      // expose top-N hypotheses via event.results[i][j]
```

### Continuous vs. Single-Shot

| Use case                             | `continuous` |
|--------------------------------------|:------------:|
| Ongoing voice control interface      | `true`       |
| One-time voice search / form fill    | `false`      |
| Push-to-talk (hold button)           | `false`      |

---

## Extending Commands

### DOM Click (fuzzy element matching)

```js
{
  pattern: /\bclick\s+(.+)/i,
  handler: (m) => {
    const label = m[1].trim().toLowerCase();
    const els = [...document.querySelectorAll('button, a, [role=button]')];
    const target = els.find(el => el.textContent.trim().toLowerCase().includes(label));
    target ? target.click() : speak(`Couldn't find "${label}"`);
  }
}
```

### Text-to-Speech Feedback (SpeechSynthesis)

```js
function speak(text, lang = 'en-US') {
  const utt = new SpeechSynthesisUtterance(text);
  utt.lang = lang;
  speechSynthesis.speak(utt);
}
```

### Form Fill

```js
{
  pattern: /\bfill\s+(.+?)\s+with\s+(.+)/i,
  handler: (m) => {
    const fieldName = m[1].trim().toLowerCase();
    const value     = m[2].trim();
    const inputs = [...document.querySelectorAll('input, textarea')];
    const target = inputs.find(el =>
      (el.placeholder || el.name || el.id || el.ariaLabel || '')
        .toLowerCase().includes(fieldName)
    );
    if (target) { target.value = value; target.dispatchEvent(new Event('input')); }
  }
}
```

---

## Permissions & UX Requirements

1. **User gesture required** — `recognition.start()` must be called inside a click/keydown handler; autostart on page load will be blocked.
2. **Microphone permission** — browser will prompt on first use; permission is remembered per origin.
3. **HTTPS required** — `SpeechRecognition` is blocked on plain `http://` (except `localhost`).
4. **Always show a status indicator** — users must know when the mic is active (legal + UX).

---

## Browser Compatibility

| Browser         | Support          |
|-----------------|-----------------|
| Chrome 33+      | ✅ Full          |
| Edge 79+        | ✅ Full          |
| Safari 14.1+    | ✅ Partial (no `continuous`) |
| Firefox         | ❌ Not supported |
| Mobile Chrome   | ✅               |
| Mobile Safari   | ⚠ Limited       |

**Always include a fallback message** if `SpeechRecognition` is undefined.

---

## Common Errors & Fixes

| `event.error`       | Cause                               | Fix                                      |
|---------------------|-------------------------------------|------------------------------------------|
| `not-allowed`       | Mic permission denied               | Prompt user to allow mic; show guidance  |
| `no-speech`         | Silence timeout                     | Restart recognition or notify user       |
| `network`           | Browser voice engine call failed    | Retry; note that some engines are online |
| `aborted`           | `recognition.abort()` was called    | Expected — no action needed              |
| `audio-capture`     | No mic found                        | Check hardware; show error UI            |

---

## See Also

- `references/speech-recognition-api.md` — Full MDN-style attribute/event reference
- `references/locales.md` — BCP 47 language codes for `recognition.lang`

For complex multi-modal apps (voice + canvas / voice + WebRTC), create a `scripts/` directory and split logic into modules.
