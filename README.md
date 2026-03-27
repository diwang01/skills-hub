# Skills Hub

**A collection of production-ready skills for OpenCode / Claude Code**

[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e?style=flat-square)](LICENSE)
[![OpenCode Skills](https://img.shields.io/badge/OpenCode-Skills-7c3aed?style=flat-square)](https://opencode.ai)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-ec4899?style=flat-square)](#contributing)

---

## What is this?

**Skills Hub** is a curated collection of skills for [OpenCode](https://opencode.ai) that enhance AI-assisted development workflows. Each skill provides domain-specific instructions, checklists, and best practices that are loaded on-demand to minimize context window usage.

---

## Available Skills

| Skill | Description | Guides |
|-------|-------------|--------|
| [**code-review**](skills/code-review/) | Comprehensive code review guidance for Java applications. Covers architecture, performance, and security. | 4 reference guides |

---

## Repository Structure

```
skills-hub/
│
├── README.md
│
└── skills/
    └── code-review/
        ├── SKILL.md                          # Core skill (~190 lines)
        └── reference/
            ├── java.md                       # Java 17/21 & Spring Boot 3
            ├── architecture-review-guide.md  # SOLID, anti-patterns, coupling
            ├── performance-review-guide.md   # N+1, complexity, memory
            └── security-review-guide.md      # OWASP Top 10, auth, validation
```

---

## Installation

**Option 1: Clone the entire repository**

```bash
git clone https://github.com/your-org/skills-hub.git
```

**Option 2: Copy individual skills**

```bash
# macOS / Linux
cp -r skills-hub/skills/code-review ~/.opencode/skills/code-review

# Or symlink
ln -s /path/to/skills-hub/skills/code-review ~/.opencode/skills/code-review
```

---

## Usage

Once installed, skills activate automatically based on your prompts:

| Prompt | Skill Activated |
|--------|-----------------|
| `Review this PR` | code-review |
| `Code review this Java class` | code-review |
| `Check this code for security issues` | code-review |
| `Architecture review` | code-review |

---

## Skills Overview

### Code Review

Transform code reviews from gatekeeping to knowledge sharing through constructive feedback, systematic analysis, and collaborative improvement.

**Key Features:**
- Four-phase review process (Context → High-Level → Line-by-Line → Summary)
- Severity labeling (`blocking` · `important` · `nit` · `suggestion` · `learning` · `praise`)
- Security-first approach based on OWASP Top 10
- Collaborative tone — questions over commands

**Supported:**
- Java 17/21 + Spring Boot 3
- Architecture design review (SOLID, anti-patterns)
- Performance review (N+1, algorithm complexity)
- Security review (auth, input validation, cryptography)

---

## Contributing

Contributions are welcome! Here are some ideas:

- **New skills** — Add skills for testing, documentation, refactoring, etc.
- **New language guides** — Python, Go, Rust, TypeScript, etc.
- **Framework guides** — Django, NestJS, Rails, etc.
- **Improvements** — Better checklists, more examples, translations

### Adding a New Skill

1. Create a folder under `skills/` with your skill name
2. Add a `SKILL.md` with frontmatter (name, description, allowed-tools)
3. Add reference guides in a `reference/` subfolder
4. Update this README

---

## License

MIT

---

Made with care for developers who value quality
