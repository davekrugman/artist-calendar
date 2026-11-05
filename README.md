# Artist Calendar — Claude Code Skill

A blueprint for building a self-hosted scheduling page that replaces Calendly.

## What this skill does

When triggered in Claude Code, it walks the user through building a custom
booking framework on their own site:

- Self-hosted booking page (no third-party iframes)
- Request → host approves → confirmation flow
- One-click approve/reject from the host's inbox
- Auto-generated unique Google Meet links per booking
- Atomic slot claiming (no double-bookings)
- `.ics` calendar invites for visitors
- Admin dashboard for OAuth connection + clear-all

## What this skill is NOT

- Not a styled component library — bring your own design system
- Not a SaaS — this builds something the user owns end-to-end
- Not for embedding existing schedulers (Calendly, Cal.com)

## Install

Clone it straight into your Claude Code skills folder:

```bash
git clone https://github.com/davekrugman/artist-calendar ~/.claude/skills/artist-calendar
```

Or download the ZIP from GitHub and copy the folder to
`~/.claude/skills/artist-calendar`.

Restart Claude Code if it was running. The skill will surface when you say
phrases like "build me a custom Calendly", "artist calendar", "self-hosted
scheduling page", etc.

## Triggers

- "artist calendar"
- "custom calendly"
- "scheduling page"
- "booking framework"
- "private booking system"
- "self-hosted scheduler"
- "open windows" (the original name when this was built for Dave Krugman)

## Stack assumptions

The skill assumes (and prompts you to adapt if needed):

- Next.js App Router
- Vercel + Vercel KV (or Upstash Redis)
- Resend for email
- Google Calendar API for Meet link generation
- TypeScript

## Credits

Originally built in a single Claude Code session for
[davekrugman.com](https://davekrugman.com)'s "Open Windows" booking page —
a 15-minute live walkthrough of his WINDOWS photo project for collectors.
Distilled into a reusable skill so other artists (or anyone) can have the
same setup on their own site.

## License

MIT. See [LICENSE](LICENSE).
