# Mobile accessibility lessons

A working checklist of hard-won TalkBack/VoiceOver accessibility lessons, distilled from
building and shipping three real Android/Kotlin Multiplatform apps with and for a blind
developer:

- **Uptimer** — a vibration-only wait-time timer (Jetpack Compose)
- **DirSpeed** — a GPS speed/direction app (Compose + a SwiftUI port)
- **Memorecorder** — a Milestone-112-inspired memo recorder (Kotlin Multiplatform, native UI
  per platform)

Every item in [`SKILL.md`](./SKILL.md) is a real bug or near-miss that was actually found —
usually only by running the app on a real device with a screen reader, not by reading the code —
not a generic accessibility tutorial. It covers things like:

- Why TalkBack's own default focus wins the race against your app's intended initial focus,
  and how to reclaim it (including on every foreground return, not just cold start)
- Why `MaterialTheme(colorScheme = ...)` alone paints nothing, and dark mode can silently do
  nothing even though the code looks completely correct
- Why a duration formatted as `"15:52"` gets read aloud as a *clock time* ("8 minutes to four"),
  not a duration — and the fix
- Merged semantics for toggle rows, text field pitfalls (multi-line-by-default, the
  select-vs-clear-on-focus race), live-region chatter, and more

## Using this as a Claude Code skill

This repo *is* a [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) — drop
`SKILL.md` into a `mobile-accessibility-lessons/` folder under your skills directory (e.g.
`~/.claude/skills/mobile-accessibility-lessons/SKILL.md` for a user-level skill, or
`.claude/skills/mobile-accessibility-lessons/SKILL.md` in a specific project) and Claude Code
will offer to load it automatically when it's relevant — building or debugging any mobile UI
that needs to work well with a screen reader.

It's plain Markdown, though, so it's just as useful as a manual checklist or review reference
even without Claude Code.

## Contributing

This is meant to grow. If you hit a TalkBack/VoiceOver bug that isn't covered here, or have a
cleaner fix for one that is, a PR or issue is very welcome — the more real, on-device-verified
lessons this collects, the more useful it gets for everyone building accessible mobile apps.

## License

MIT — see [LICENSE](./LICENSE).
