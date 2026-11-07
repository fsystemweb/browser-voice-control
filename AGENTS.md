# AGENTS.md

Content-only Agent Skills package: a `SKILL.md` that teaches agents to build browser voice-control interfaces with the native Web Speech API. No code to build or run.

## Commands
- Install (as a user of the skill, per `README.md`): `mkdir -p .claude/skills/browser-voice-control` then `cp -r browser-voice-control/. .claude/skills/browser-voice-control/` inside a target project
- Alternative install from README: `npx agent-skills-cli add fsystemweb/browser-voice-control`
- Repo itself: no package manifest, no dependencies
- Build, lint, tests: none yet
- Manual check: open `browser-voice-control/references/best-practices-example.html` in Chrome or Edge, served from `localhost` or HTTPS

## Conventions
- `browser-voice-control/SKILL.md`: the skill itself. It starts with YAML frontmatter (`name`, `description`, `compatibility`); the `description` is the trigger text, so keep it accurate.
- `browser-voice-control/references/`: `speech-recognition-api.md` (API reference), `locales.md` (BCP 47 codes), `best-practices-example.html` (self-contained boilerplate).
- `README.md`: install guide, supported commands, browser support and locale tables. Keep it consistent with `SKILL.md`.
- `skill-speech.jpg`: README header image.
- Everything runs client-side with no API keys and no third-party libraries; keep examples dependency-free.
- Browser support claims (Chrome, Edge, Safari partial, Firefox unsupported) appear in both `SKILL.md` and `README.md`; update both together.
- Folder `browser-voice-control/` is the skill folder; its name must match the `name` field in `SKILL.md`.

## Boundaries
- Always: update `README.md` when the command list, browser support or file structure in `SKILL.md` changes.
- Ask first: renaming the skill folder or the `name` field, adding files that need a build step or dependency.
- Never: commit secrets or read `.env` files.
- Never: add example code that calls external APIs, `fetch`es remote services or needs an API key.
