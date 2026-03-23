# Agent Skills

A collection of custom skills for Claude Code, focused on power electronics simulation automation.

## Skills

| Skill | Description |
|---|---|
| [plecs-sim-autotune](plecs-sim-autotune/) | Automate PLECS power converter simulation and PID auto-tuning via XML-RPC. Covers Type 3 compensator design, transient response analysis, and rule-based 2D parameter search. |
| [plecs-rpc-scope-export](plecs-rpc-scope-export/) | Export PLECS scope data (CSV/bitmap) through RPC without relying on the GUI. Includes MATLAB plotting template. |

## Usage

To use a skill in your project, copy the skill folder into your project's `.claude/skills/` directory:

```bash
cp -r plecs-sim-autotune /path/to/your/project/.claude/skills/
```

Claude Code will automatically detect and invoke the skill based on your request context.

## Structure

```
agent-skills/
├── README.md
├── plecs-sim-autotune/
│   └── SKILL.md
└── plecs-rpc-scope-export/
    └── SKILL.md
```
