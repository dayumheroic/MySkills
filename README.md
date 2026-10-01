# MySkills

A collection of Agent Skills and plugins kept under our control, with review notes.

| Folder | What it is | Review |
|---|---|---|
| `claude-video/` | The **watch** skill (v0.3.2) from bradautomates/claude-video. Gives a coding agent a video input: frames plus transcript locally, or a Gemini video-model answer with a key. Skill lives at `claude-video/skills/watch/`. | [reviews/claude-video-review.md](reviews/claude-video-review.md) |

## Installing the watch skill from this repo

Claude Code (inside a session):

```text
/plugin marketplace add dayumheroic/MySkills
```

The plugin manifest lives at `claude-video/.claude-plugin/`, so for Claude Code's
marketplace use the subfolder, or install the skill folder directly with the Skills CLI:

```bash
npx skills add dayumheroic/MySkills --skill claude-video/skills/watch -g
```

Or copy `claude-video/skills/watch/` whole into your agent's skills directory.
Do not split `SKILL.md` from its sibling `scripts/`.
