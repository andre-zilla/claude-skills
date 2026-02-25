# Claude Code Custom Skills

A collection of custom [Claude Code](https://claude.ai/claude-code) skills for content creation, AI development, security auditing, and community management.

These skills are designed for tech content creators, AI developers, and security professionals. Install them into your `~/.claude/skills/` directory and invoke them with `/skill-name` inside Claude Code.

---

## Quick Start

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/andre-zilla/claude-skills.git ~/.claude/skills

# That's it — open Claude Code and type / to see all available skills
```

**Requirements:** [Claude Code CLI](https://claude.ai/claude-code) installed and configured.

---

## Skills Overview

| Skill | Category | Description |
|---|---|---|
| [`/content-plan`](#content-plan) | Content | Weekly content calendar generator |
| [`/script-writer`](#script-writer) | Content | Video & blog script generator |
| [`/seo-optimize`](#seo-optimize) | Content | YouTube & blog SEO optimizer |
| [`/social-repurpose`](#social-repurpose) | Content | Multi-platform content repurposer |
| [`/tool-review`](#tool-review) | Content | AI tool & tech product review generator |
| [`/project-ideas`](#project-ideas) | Development | AI/tech project idea generator |
| [`/app-scaffold`](#app-scaffold) | Development | Project scaffolding with latest versions |
| [`/secure-review`](#secure-review) | Security | Comprehensive security audit |
| [`/stack-check`](#stack-check) | Security | Dependency & version auditor |
| [`/community-manager`](#community-manager) | Community | Bug bounty community management assistant |

---

## Content Creation Skills

### `/content-plan`

Generates a 7-day content calendar tailored for tech and AI content creators.

**Usage:**
```
/content-plan AI tools and workflows
/content-plan Claude Code tutorials
```

**What it produces:**
- Daily content assignments across YouTube, Twitter/X, LinkedIn, Blog, TikTok/Reels, and Newsletter
- SEO-friendly titles and attention-grabbing hooks
- Key talking points (3-5 per post)
- Calls to action and estimated production time
- Optimal posting times per platform
- Weekly theme summary tying all content together

**Rules it follows:**
- Mixes platforms — never posts to the same platform 3 days in a row
- Includes at least 1 quick-win post and 1 long-form deep dive per week
- Prioritizes trending AI/tech topics

---

### `/script-writer`

Creates ready-to-use scripts for multiple content formats.

**Usage:**
```
/script-writer Building an AI Agent youtube
/script-writer Top 5 AI Tools 2026 short
/script-writer Prompt Engineering Guide blog
/script-writer Claude Code Tips thread
```

**Supported formats:**
| Format | Output |
|---|---|
| `youtube` (default) | 5-15 min video script with timestamps, B-roll cues, thumbnail ideas |
| `short` | 30-60 sec TikTok/Reels/Shorts script with text overlay suggestions |
| `blog` | 800-1500 word blog post with headline, H2 sections, meta description |
| `thread` | 8-12 tweet Twitter/X thread with hook and CTA |

**Includes:** `[B-ROLL]` and `[SCREEN RECORDING]` cues, SEO keywords, suggested timestamps, thumbnail concepts.

---

### `/seo-optimize`

Optimizes content for YouTube search ranking and blog SEO.

**Usage:**
```
/seo-optimize "Building an AI App with Claude" youtube
/seo-optimize "Getting Started with Prompt Engineering" blog
```

**YouTube optimization:**
- 3 title variants (under 60 chars, front-loaded keywords, power words)
- Optimized description with keyword placement
- 15-20 tags (broad + specific + misspellings)
- Thumbnail text suggestions and hashtags

**Blog optimization:**
- H1 title with primary keyword
- Meta description (150-160 chars)
- Clean URL slug
- Header structure with keyword-rich H2/H3 suggestions
- Schema markup recommendations

**Both:** Primary keyword with volume estimate, secondary keywords, long-tail variations, trending angles, and competitor gap analysis.

---

### `/social-repurpose`

Transforms one piece of long-form content into posts for 5 platforms.

**Usage:**
```
/social-repurpose ./blog-post.md
/social-repurpose [paste content directly]
```

**Generates:**
| Platform | Format |
|---|---|
| Twitter/X | 8-12 tweet thread with hook and CTA |
| LinkedIn | Professional post with hook line, short paragraphs, hashtags |
| Instagram/TikTok | Caption + 20-30 hashtags + carousel/visual ideas |
| YouTube Shorts/TikTok | 30-60 sec script with text overlay suggestions |
| Newsletter | 3 subject lines + preview text + 150-word summary |

Each version feels native to its platform — not copy-pasted. Includes posting time suggestions and stagger schedule.

---

### `/tool-review`

Generates structured review outlines for AI tools and tech products.

**Usage:**
```
/tool-review Cursor IDE
/tool-review Claude Code
/tool-review Midjourney v7
```

**Produces:**
- **Overview:** What it is, who it's for, category, pricing tiers
- **Key Features:** Top 5-7 with standout differentiators
- **Pros & Cons:** 3-5 each, specific and honest
- **Hands-On Demo Ideas:** Scenarios to try on camera, workflows to test, "wow moments"
- **Competitors:** Top 2-3 alternatives with comparison
- **Verdict:** Rating/10, "Best for..." and "Skip if..." one-liners
- **Video Talking Points:** 5-7 bullets + thumbnail concept + 3 title options

Always uses web search to get the latest pricing, features, and updates.

---

## Development Skills

### `/project-ideas`

Generates buildable project ideas with content angles and monetization potential.

**Usage:**
```
/project-ideas
/project-ideas intermediate nextjs AI
/project-ideas beginner python automation
```

**Per idea (5 total):**
- Catchy project name and one-liner
- Full tech stack (latest versions, verified via web search)
- 4-6 core features and required APIs/services
- Difficulty level and build time estimate
- **Content angle:** Video series potential, wow factor, teachable moments
- **Monetization:** SaaS potential, open source with sponsors, product viability

Always includes at least one weekend project (ship fast) and one ambitious project (long-form content series).

---

### `/app-scaffold`

Scaffolds a complete project with the latest stable versions of everything.

**Usage:**
```
/app-scaffold AI chatbot with Next.js and Supabase
/app-scaffold SaaS dashboard with Stripe billing
/app-scaffold CLI tool in Rust
```

**Creates:**
- Clean folder structure following framework conventions
- `CLAUDE.md` — Project context for Claude Code
- `README.md` — Setup instructions with version numbers
- `package.json` (or equivalent) — Dependencies pinned to latest stable
- `.gitignore` — Comprehensive for the stack
- Base app code — Minimal working app with routing and example feature
- Config files — ESLint, Prettier, TypeScript as appropriate

**Security defaults baked in:**
- Environment variables for all secrets (`.env.example` included)
- CORS configuration
- Input validation setup
- CSP headers for web apps
- Auth boilerplate when applicable

**Always verifies latest versions via web search** — never uses stale cached versions.

---

## Security Skills

### `/secure-review`

Runs a comprehensive, layered security audit on your project.

**Usage:**
```
/secure-review
/secure-review src/api/
/secure-review auth-module
```

**Three-layer approach:**

| Layer | What It Does |
|---|---|
| **1. Automated Scans** | Runs Ghost Security scans: secrets detection, dependency CVEs, SAST code analysis |
| **2. Manual Code Review** | OWASP Top 10 review: auth/authz, injection flaws, data protection, API security, configuration |
| **3. Dependency Health** | Flags outdated, deprecated, and supply-chain-risk packages |

**Output format:**
- **Executive Summary** — Overall risk level + top 3 immediate actions
- **Detailed Findings** — Severity, OWASP category, file:line, description, impact, remediation with code examples
- **Remediation Priority List** — Ordered checklist, highest priority first

**Note:** Requires the [Ghost Security plugin](https://github.com/anthropics/claude-code-plugins) for automated scanning. Manual review works without it.

---

### `/stack-check`

Audits every dependency, framework, and runtime version in your project.

**Usage:**
```
/stack-check
/stack-check ./my-app
```

**Detects and audits:**
- Node.js (package.json, lockfiles)
- Python (requirements.txt, pyproject.toml, poetry.lock)
- Go (go.mod, go.sum)
- Ruby (Gemfile, Gemfile.lock)
- Rust (Cargo.toml, Cargo.lock)
- Docker (base images in Dockerfile/docker-compose)
- Framework configs (next.config.js, vite.config.ts, etc.)
- Runtime versions (Node.js, Python, Go, Rust themselves)

**Classification for each package:**
| Status | Meaning |
|---|---|
| Current | On latest stable |
| Minor Behind | Patch or minor update available |
| Major Behind | Major version behind — action needed |
| Deprecated | Package is deprecated or abandoned |
| EOL | End of life — no security patches |
| Vulnerable | Has known CVEs |

**Produces:** Summary table, critical/recommended/optional update lists, and a migration guide with breaking changes and suggested upgrade order.

**Always web searches** for latest versions — never relies on cached knowledge.

---

## Community Management

### `/community-manager`

Comprehensive community management assistant built for the [0DIN.ai](https://0din.ai) GenAI bug bounty platform. Adaptable for any bug bounty or security researcher community.

**Usage:**
```
/community-manager
/community-manager blog ideas for March
/community-manager plan a CTF event
/community-manager re-engagement campaign for dormant researchers
/community-manager monthly metrics report template
```

**7 focus areas:**

| Focus Area | What It Generates |
|---|---|
| **Engagement Strategies** | Activation campaigns, retention plays, re-engagement sequences, referral programs, feedback loops |
| **Blog & Content Ideas** | Researcher spotlights, vulnerability deep dives, how-to guides, program updates, beginner pathways, expert interviews |
| **Community Activities** | CTF challenges, Hack-The-Model sprints, live hacking sessions, AMAs, workshops, bug bash weeks, mentor matching |
| **Onboarding & Growth** | Signup-to-submission funnels, skill progression paths, university outreach, cross-pollination strategies |
| **Recognition & Rewards** | Achievement ideas, researcher of the month, leaderboard campaigns, exclusive access, swag campaigns, career development |
| **Communication** | Announcement templates, weekly digests, researcher outreach DMs, social cadence, Discord engagement, crisis playbooks |
| **Metrics & Reporting** | Community health dashboard, engagement/pipeline/quality/retention KPIs, monthly report templates |

When invoked without a focus area, generates a **weekly community action plan** covering top priorities across all areas.

---

## Installation

### Fresh Install
```bash
git clone https://github.com/andre-zilla/claude-skills.git ~/.claude/skills
```

### Already Have Skills
```bash
# Back up existing skills
cp -r ~/.claude/skills ~/.claude/skills-backup

# Clone and merge
git clone https://github.com/andre-zilla/claude-skills.git /tmp/claude-skills
cp -r /tmp/claude-skills/*/ ~/.claude/skills/
rm -rf /tmp/claude-skills
```

### Sync Across Machines
```bash
# On any machine after initial clone
cd ~/.claude/skills
git pull
```

### Verify Installation
Open Claude Code and type `/` — you should see all skills in the autocomplete menu.

---

## Customization

Each skill is a standalone `SKILL.md` file with YAML frontmatter. To customize:

1. Edit the `SKILL.md` in any skill directory
2. Modify the instructions, rules, or output format
3. Changes take effect immediately — no restart needed

### Creating Your Own Skills
```bash
mkdir ~/.claude/skills/my-skill
```

Create `SKILL.md` with this structure:
```yaml
---
name: my-skill
description: What this skill does and when to use it.
argument-hint: "[optional arguments]"
---

Your instructions for Claude here...
```

---

## License

MIT — use, modify, and share freely.
