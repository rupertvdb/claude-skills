# claude-skills

Claude Code skills I actually use every day, published as a plugin
marketplace — each skill is its own independently installable plugin, so you
take exactly the one you want and nothing else.

## Install

```
/plugin marketplace add rupertvdb/claude-skills
/plugin install adversarial@claude-skills
```

That's it. Type `/adversarial` in any Claude Code session.

## What's in here

| Plugin | What it does |
|---|---|
| [`adversarial`](plugins/adversarial/skills/adversarial/SKILL.md) | `/adversarial` — dispatches a fresh, hostile, **read-only** reviewer against the code or plan your session just produced. The reviewer runs in an isolated subagent with a cold context (it never sees the thread's rationalisations) and is pinned to a strong model (Opus by default), so it can't be talked into liking the work. Report-only: it returns ranked findings (BLOCKER / MAJOR / MINOR, each with a concrete failure scenario) and a SHIP / FIX-FIRST / RETHINK verdict. You decide what to act on. |

More coming — each future skill lands as its own plugin in `plugins/`.

## Why /adversarial

Self-review fails for a predictable reason: the context that produced the work
also contains all the reasoning that convinced you it was right. A reviewer
reading the same thread inherits that anchoring. `/adversarial` breaks the
anchor two ways: the reviewer starts **cold** (fresh context, sees only the
artifact), and it's **framed hostile** ("assume it is wrong until the code
proves otherwise"). In practice this catches the class of bug the author is
structurally blind to — the hidden assumption, the unhandled partial failure,
the auth gap that "obviously" couldn't happen.

## Is this safe to install?

Reasonable question to ask of anything you add to your agent. Here's exactly
what you're installing:

- **A skill is a plain markdown file of instructions** — read the whole thing
  before installing: [SKILL.md](plugins/adversarial/skills/adversarial/SKILL.md).
  There is **no executable code, no hooks, no scripts, no MCP servers** in this
  repo. Nothing runs on install.
- A skill only *instructs* Claude; every action Claude then takes still goes
  through Claude Code's normal permission prompts on your machine.
- This particular skill is deliberately **read-only by design**: the reviewer
  it dispatches is explicitly forbidden from editing files, and the skill ends
  at a report — it never applies fixes.

## License

MIT — see [LICENSE](LICENSE).
