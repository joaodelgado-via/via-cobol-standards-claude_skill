# via-cobol — Claude Skill

A Claude skill that enforces **VIA Consulting's COBOL / IBM i coding standards** on every piece of generated code.

## What it does

When installed, Claude automatically applies VIA's conventions whenever anyone asks it to write, generate, or modify COBOL code — no extra prompting required. This covers:

- Object naming (8-character, application-code + type + identifier)
- Program structure (1000-INICIO / 2000-PROCESSA / 3000-FIM)
- Working-storage prefixes (`WS-`, `SW-`, `CON-`) and `VALUE` initialisation
- Single quotes throughout (mainframe portability)
- `PERFORM … THRU …-EXIT` paragraph invocation pattern
- Guard pattern (`IF AQCW0009-SEM-ERROS` / `IF VIAL0002-SEM-ERROS`)
- Batch mandatory frame (AQCP0100, AQCP0009, AQCP0002)
- Online director pattern (VIAL0001 / VIAL0002 / VIA00100)
- Per-resource DB2 status variables with 88-level conditions
- Module CALL pattern (ETCL0002 first, ON EXCEPTION, ETCP0003 pre/post)
- Debug writes to VIAF0015 via VIAP0016 at every paragraph entry
- One action per paragraph (one SQL / one CALL / one validation)

## Repository structure

```
via-cobol-skill/
├── SKILL.md          # The skill loaded by Claude — all rules live here
├── evals/
│   └── evals.json    # Test prompts used to validate the skill
└── README.md         # This file
```

## Installation

1. Download the latest `via-cobol.skill` file from [Releases](../../releases).
2. Open **Claude desktop** (Cowork mode).
3. Click the skill file — a "Save skill" button will appear.
4. Click **Save skill**.

Claude will now apply VIA COBOL standards automatically in every session.

## Updating the skill

Edit `SKILL.md` directly — it is the single source of truth. After editing:

1. Repackage: zip the `via-cobol-skill/` folder and rename the `.zip` to `.skill`.
2. Distribute the new `.skill` file to the team.

## Running the evals

The `evals/evals.json` file contains the test prompts used during development. To re-run them after making changes to `SKILL.md`, use the **skill-creator** skill in Claude.

## Standards reference

The full VIA COBOL Quick Reference Card that this skill is based on is maintained separately by the team. The rules encoded in `SKILL.md` reflect the version agreed upon at initial skill creation (May 2026).
