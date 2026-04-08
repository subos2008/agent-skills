# agent-skills

Personal collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) skills, packaged as an installable plugin.

## Skills included

| Skill | Purpose |
|-------|---------|
| `spa-aws-deploy` | Set up AWS deployment infrastructure for React/Vite SPAs backed by Supabase: Terraform modules (S3, CloudFront, Route 53, ACM), per-app + smart parallel deploy scripts, and a GitHub Actions CI/CD pipeline. |

## Install

In any Claude Code session:

```
/plugin marketplace add subos2008/agent-skills
/plugin install agent-skills@agent-skills
```

That's it. All skills in `skills/` are auto-loaded by Claude.

To update later, after pushing changes:

```
/plugin marketplace update agent-skills
```

### Private clone (machines with SSH keys configured)

If this repo were private, the same commands work as long as the machine has git/SSH credentials that can clone `git@github.com:subos2008/agent-skills.git`. For background auto-updates on private repos, set `GITHUB_TOKEN` in the environment.

## Repo layout

```
agent-skills/
├── .claude-plugin/
│   ├── plugin.json         # Plugin metadata
│   └── marketplace.json    # Single-plugin marketplace pointing at "."
├── skills/
│   └── spa-aws-deploy/
│       ├── SKILL.md
│       └── references/
│           ├── terraform.md
│           ├── deploy-scripts.md
│           └── github-actions.md
├── LICENSE
└── README.md
```

## Adding more skills

1. Create a new directory under `skills/<new-skill-name>/`
2. Add `SKILL.md` with frontmatter (`name`, `description`)
3. Optionally add `references/`, `scripts/`, or `assets/` subdirectories
4. Bump the version in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
5. Commit, push, then on each machine: `/plugin marketplace update agent-skills`

The plugin auto-discovers any skill folder under `skills/` — no marketplace.json edit needed unless you want to bump the version.

## License

MIT
