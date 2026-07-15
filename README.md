# docs-skill

A skill to compile docs into a more readable & targeted format, exported as a `.md`.

## Layout

We each iterate on our own configurations so we never edit the same file.

```
docs-skill/
├── fixtures/          # SHARED input docs — the constant across every experiment
├── branson/
│   ├── v1/SKILL.md
│   └── v2/SKILL.md
├── ryan/
│   ├── v1/SKILL.md
│   └── v2/SKILL.md
└── outputs/           # generated .md, mirrored per person/version, git-tracked
    ├── branson/{v1,v2}/
    └── ryan/{v1,v2}/
```

## Rules of the road
- **Own your folder.** Edit only under your own name. This is how we avoid conflicts.
- **Fix the input.** Every version runs against the *same* docs in `fixtures/`. If inputs differ, you're comparing docs, not configs.
- **Commit outputs.** So we can `git diff` results and compare versions honestly.
- **Promote a winner.** When a config wins, copy it to a top-level `compile-docs/` and retire the rest.
