# Agent Skills

A collection of reusable custom skills for OpenClaw / Claude Code style workflows, including OpenClaw maintenance, ChatGPT OAuth integration, and PLECS automation.

## Skills

| Skill | Description |
|---|---|
| [chatgpt-oauth-bridge](chatgpt-oauth-bridge/) | Add ChatGPT OAuth / OpenAI Codex OAuth support to an existing project. Covers browser login, local credential storage, dynamic codex model discovery, runtime bridging, tool-call handling, and CLI integration. |
| [openclaw-control-ui-fix](openclaw-control-ui-fix/) | Fix missing OpenClaw Control UI assets after upgrades when `openclaw gateway` reports missing `dist/control-ui/index.html`. |
| [plecs-sim-autotune](plecs-sim-autotune/) | Automate PLECS power converter simulation and PID auto-tuning via XML-RPC. Covers Type 3 compensator design, transient response analysis, and rule-based 2D parameter search. |
| [plecs-rpc-scope-export](plecs-rpc-scope-export/) | Export PLECS scope data (CSV/bitmap) through RPC without relying on the GUI. Includes MATLAB plotting template. |

## Usage

To use a skill in your project, copy the skill folder into your project's skill directory.

Example:

```bash
cp -r chatgpt-oauth-bridge /path/to/your/project/.claude/skills/
```

If you use OpenClaw workspace-local skills, you can also copy skills into a local `skills/` directory and reuse them across projects.

## Structure

```text
agent-skills/
├── README.md
├── chatgpt-oauth-bridge/
│   ├── SKILL.md
│   └── references/
├── openclaw-control-ui-fix/
│   └── SKILL.md
├── plecs-rpc-scope-export/
│   └── SKILL.md
└── plecs-sim-autotune/
    └── SKILL.md
```
