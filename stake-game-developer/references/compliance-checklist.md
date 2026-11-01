# Compliance Checklist

## Required Gate

Run compliance checks as a release blocker:

```bash
node scripts/audit-checklist.mjs \
  --rules references/compliance-rules.json \
  --target <path> \
  --social true \
  --format text
```

## Frontend Requirements

- Main game frame is not scrollable.
- Production build does not emit console errors.
- Sound can be disabled in UI.
- Assets are loaded from expected CDN pathing.
- Mobile view remains usable.
- Double-tap zoom is disabled on mobile.

## Rules and Presentation

- Payout info and win-combo rules are clear.
- Game modes and mode costs are clear.
- Bonus/free-spin trigger conditions are clear.
- Bet-mode confirmations are present for high-cost mode changes.
- Disclaimer text required by platform is present.

## RGS and Replay

- Use `rgs_url` query parameter for API target.
- Respect RGS-provided min/max/default bet levels.
- Use RGS-provided balance values.
- Replay URL and optional replay params are supported.

## Social/Jurisdiction

- When `social=true`, restricted phrases must be replaced.
- Do not use prohibited “bet/buy/gamble” style wording in social context.
- Ensure SC/GC display rules are respected where required.

## Reading the social audit

`audit-checklist.mjs --social true` is a text scan, so three classes of hit are
expected and are not violations. Triage them before treating the count as a
blocker:

- **Code identifiers.** `symbolState === 'win'`, `import Win from …`,
  `win.positions`. The scanner cannot tell an identifier from player copy.
- **Dual-wording branches.** Games that localise terminology per mode hold both
  variants in source, e.g.
  `bet: social ? 'amount' : 'bet'`. The non-social branch will always be
  flagged even though it never renders in social mode. This is the correct
  implementation, not a defect.
- **Test fixtures.** Book/event JSON under `stories/`, `fixtures/` or similar
  is data, not copy. Point `--target` at the source directories that actually
  ship player text.

What *is* a real finding: a restricted term inside a literal string that reaches
the player unconditionally - button labels, modal prose, ticker text,
error messages.

The scanner matches whole words only and skips comment bodies, so
"glowing"/"window" no longer match "win" and developer notes are ignored.

## Fail Policy

Any confirmed compliance violation is a hard fail for release recommendation.
Triage the social audit per the section above before applying this.
