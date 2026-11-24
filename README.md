# app-perf-audit

An agent skill for **device-measured app performance audits**: find out why an app is slow, dark-screened, battery-hungry or crashing on other people's phones — with evidence instead of intuition.

Most performance advice on the internet is a list of generic tips ("use a virtualized list", "memoize your components"). This skill is built the other way around: it starts from the **symptom** the user reports, names the concrete causes that actually produce it, and requires **the same measurement before and after** any fix.

## Why it exists

Written after a real audit where:
- a two-line dead branch made a screen remount its whole tree on every reopen;
- a pan gesture on a child swallowed the scroll for the entire screen;
- a native modal inside another native modal produced a dimmed screen with dead input;
- infinite looping animations kept the app at ~50 fps while producing zero changed pixels;
- the "fix" that moved work to the first keystroke made search take six seconds — and had to be reverted the same session.

None of those are in generic "best practices" lists. They are in `references/symptoms.md`.

## What's inside

```
skills/app-perf-audit/
  SKILL.md                  workflow, hard rules, symptom index
  references/symptoms.md    symptom → cause, with grep hints and a cross-platform map
  references/recipes.md     verified fix patterns with code
  references/measure.md     adb protocol: cold start, jank, memory, weak-device profile
  references/process.md     process traps: hook order, build OOM, device focus, maintenance
```

## Install

Agent Skills are just a folder with a `SKILL.md` (YAML frontmatter + Markdown). Copy the skill folder into your agent's skills directory:

- **OpenCode (project scope)**: `.opencode/skills/app-perf-audit/`
- **OpenCode (user scope)**: `~/.config/opencode/skills/app-perf-audit/`
- **Claude Code**: `~/.claude/skills/app-perf-audit/`

No dependencies, no scripts to run.

## Use

Ask for what you actually observe:

- "the app takes a second to open the moon screen"
- "scrolling stutters"
- "it just dims the screen and nothing appears"
- "it's fine on my phone, but users complain"
- "it gets warm / drains battery on Home"

The skill leads with detection, then measurement, then the smallest complete fix.

## Scope and limitations

- Built from **React Native / Android** sessions; the recipes are concrete there.
- The cross-platform map (Compose, Flutter, SwiftUI, Qt/QML) is **conceptual**, not verified recipe by recipe. Treat it as a starting point, not as tested guidance.
- It has been validated on **one** real product and on a physical device plus a low-end emulator. It is not a benchmark suite.
- It does not replace profiling tools; it tells you where to point them.

## Contributing

If an audit finds a cause that is not in `symptoms.md`, add it there (symptom → cause → grep hint). If a recipe proves itself on a device, add the measured effect to `recipes.md`. A recipe with no measured effect is a hypothesis — label it as one.

## License

MIT — see [LICENSE](LICENSE).
