# Review Prompts for AI-Assisted Code Review

AI-assisted code review prompts for Linux kernel, barebox, systemd, and iproute
development.  Works with Claude Code and other AI tools.

## Quick Start

### Install Prompts

```bash
./setup.sh <agent> <project>
```

Where `<agent>` is one of available agents and `<project>` is one of available
projects that are explicitly stated in the usage message when the script is
executed with `-h|--help` option.

## Available Commands

| Project | Review | Debug | Verify |
|---------|--------|-------|--------|
| Kernel | `/kreview` | `/kdebug` | `/kverify` |
| barebox | `/breview` | `/bdebug` | `/bverify` |
| systemd | `/systemd-review` | `/systemd-debug` | `/systemd-verify` |
| iproute | `/iproute-review` | `/iproute-debug` | `/iproute-verify` |

## Project Documentation

* [Kernel Review Prompts](kernel/README.md) - Linux kernel specific patterns and protocols
* [barebox Review Prompts](barebox/README.md) - an overlay on the kernel prompts, carrying only what differs in barebox
* [systemd Review Prompts](systemd/README.md) - systemd specific patterns and protocols
* [iproute Review Prompts](iproute/README.md) - iproute specific patterns and protocols

## How It Works

Each project has:
- **Skill file** - Automatically loads context when working in the project tree
- **Slash commands** - Quick access to review, debug, and verify workflows
- **Subsystem files** - Domain-specific knowledge loaded on demand

The skills detect your working directory and load appropriate context:
- In a kernel tree: kernel skill loads automatically
- In a barebox tree: barebox skill loads automatically
- In a systemd tree: systemd skill loads automatically
- In an iproute tree: iproute skill loads automatically

barebox is the exception to "each project has": it is an overlay, not a
standalone set.  It carries only the barebox deltas and reads the generic
protocol from `kernel/`.

## Structure

```
review-prompts/
├── kernel/                    # Linux kernel prompts
│   ├── skills/               # Skill template
│   ├── slash-commands/       # /kreview, /kdebug, /kverify
│   ├── scripts/              # Setup script and utilities
│   ├── patterns/             # Bug pattern documentation
│   └── *.md                  # Subsystem and protocol files
│
├── barebox/                   # barebox prompts (overlay on kernel/)
│   ├── skills/               # Skill template
│   ├── slash-commands/       # /breview, /bdebug, /bverify, /bseries,
│   │                         #   /bcocci, /borcreview
│   ├── subsystem/            # barebox-specific guides + kernel guide index
│   ├── deltas.md             # Where barebox differs from Linux
│   └── *.md                  # Overlay entry points; no scripts/ here
│
├── systemd/                   # systemd prompts
│   ├── skills/               # Skill template
│   ├── slash-commands/       # /systemd-review, /systemd-debug, /systemd-verify
│   ├── scripts/              # Setup script
│   ├── patterns/             # Bug pattern documentation
│   └── *.md                  # Subsystem and protocol files
│
├── iproute/                  # iproute prompts
│   ├── skills/               # Skill template
│   ├── slash-commands/       # /iproute-review, /iproute-debug, /iproute-verify
│   ├── scripts/              # Setup script
│   ├── patterns/             # Bug pattern documentation
│   └── *.md                  # Subsystem and protocol files
│
└── README.md                  # This file
```

## Semcode Integration

These prompts work best with [semcode](https://github.com/facebookexperimental/semcode)
for fast code navigation and semantic search.

## License

See [LICENSE](LICENSE) for license information.
