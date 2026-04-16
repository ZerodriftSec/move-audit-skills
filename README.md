# Move Audit

Agent-native Move security infrastructure for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Audits Sui and Aptos smart contracts using parallel specialist agents with depth follow-up and structured validation.

## Install

In your project directory, run:

```
npx skills add https://github.com/ZerodriftSec/move-skills
```

## Architecture

```
move-skill/
├── agents/                            ← Agent definitions
│   │  Breadth agents (2, each covers a vulnerability domain)
│   ├── ability-type-safety-agent.md       abilities, generics, type constraints
│   ├── ownership-composability-agent.md   ownership models, composability
│   │
│   │  Depth agents (3, triggered by high-signal breadth findings)
│   ├── depth-token-flow-agent.md          balance inconsistencies
│   ├── depth-state-trace-agent.md         multi-function state mutations
│   └── depth-edge-case-agent.md           boundary conditions, dust
│
├── skills/
│   ├── sui-move-auditor/              ← Sui orchestrator
│   │   ├── SKILL.md
│   │   ├── scripts/
│   │   │   └── banner.sh              ← Sui Move Auditor banner
│   │   └── references/                ← 26 Sui-specific audit modules
│   │       ├── attack-vectors.md          Sui attack vector catalog
│   │       ├── object-ownership.md
│   │       ├── ptb-composability.md
│   │       ├── package-version-safety.md
│   │       └── ...
│   │
│   └── aptos-move-auditor/            ← Aptos orchestrator
│       ├── SKILL.md
│       ├── scripts/
│       │   └── banner.sh              ← Aptos Move Auditor banner
│       └── references/                ← 26 Aptos-specific audit modules
│           ├── attack-vectors.md          Aptos attack vector catalog
│           ├── reentrancy-analysis.md
│           ├── ref-lifecycle.md
│           ├── fungible-asset-security.md
│           └── ...
│
└── .claude-plugin/
    └── plugin.json
```

### Layer Breakdown

| Layer | Purpose |
|-------|---------|
| `agents/` | Specialist personas — breadth agents for domain coverage, depth agents for high-signal follow-up |
| `skills/sui-move-auditor/` | Sui-specific orchestrator with banner, 26 audit modules, Sui vulnerability catalog and attack vectors |
| `skills/aptos-move-auditor/` | Aptos-specific orchestrator with banner, 26 audit modules, Aptos vulnerability catalog and attack vectors |

### Audit Pipeline

```
Turn 0  Banner               Print platform-specific banner
Turn 1  Discover             Find .move files, load validation rules
Turn 2  Prepare              Build source bundles with specialist modules
Turn 3  Run Specialists      8 parallel breadth agents, each with dedicated skill modules
Turn 4  Depth Analysis       Triggered depth agents for high-signal findings
Turn 5  Validate & Report    Deduplicate, gate-evaluate, score, format final report
```

## Sui Vulnerability Categories

| ID | Category | Severity |
|----|----------|----------|
| S1 | Object Ownership Bypass | CRITICAL |
| S2 | Shared Object Manipulation | CRITICAL |
| S3 | PTB Composition Attacks | HIGH |
| S4 | Kiosk Exploitation | HIGH |
| S5 | Dynamic Field Abuse | HIGH |
| S6 | Transfer Policy Bypass | HIGH |
| S7 | Capability Leakage | HIGH |
| S8 | Witness Pattern Abuse | CRITICAL |
| S9 | Improper Abilities | CRITICAL |
| S10 | Upgrade Cap Mishandling | HIGH |

## Aptos Vulnerability Categories

| ID | Category | Severity |
|----|----------|----------|
| A1 | Signer Validation Bypass | CRITICAL |
| A2 | Account Resource Abuse | HIGH |
| A3 | Event Handle Manipulation | MEDIUM |
| A4 | FungibleAsset Vulnerabilities | HIGH |
| A5 | Table/SmartVector Issues | MEDIUM |
| A6 | Multi-Signature/Auth Key | MEDIUM |
| A7 | Capability Leakage | HIGH |
| A8 | Witness Pattern Abuse | CRITICAL |
| A9 | Improper Abilities | CRITICAL |
| A10 | Reentrancy (Move 2.2+) | HIGH |

## License

MIT
