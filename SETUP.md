# Setup — project-scout-subagents

The skill works with **no setup at all**. The tokens below are **optional** —
they only raise rate limits and add exact star counts / dependency-health
scores. Create each as a plain-text file in `~/.claude/` containing only the
token (no quotes, no newline fuss):

| File | Purpose | Where to get it |
|------|---------|-----------------|
| `~/.claude/github-token` | Authenticated GitHub search — exact star counts, no rate limit | GitHub → Settings → Developer settings → Personal access tokens (classic); scope `public_repo` (read) is enough |
| `~/.claude/libraries-io-key` | Libraries.io SourceRank package-health scores | libraries.io → Account settings → API Key |
| `~/.claude/stackoverflow-key` | Stack Exchange API (community-signal queries) | stackapps.com/apps/oauth/register |

Example:

```bash
printf '%s' 'ghp_xxxxxxxxxxxxxxxx' > ~/.claude/github-token
chmod 600 ~/.claude/github-token
```

The skill reads each with `cat ~/.claude/<file> 2>/dev/null`, so a missing file
just falls back to the unauthenticated path — nothing breaks. **Keep these files
private; never commit them.**

That's the whole machine-specific dependency. With (or without) these in place,
prompting Claude to scout a project reproduces the author's exact behaviour.
