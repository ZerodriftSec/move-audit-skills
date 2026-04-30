# Move TxAnalyzer Investigation

## Goal

Build a CLI that analyzes hack / suspicious on-chain transactions for Sui and Aptos Move, analogous to `BradMoonUESTC/TxAnalyzer` for EVM/BSC, but using Move-native evidence: transaction effects, object/resource changes, events, balance diffs, package/module metadata, and source/binary provenance.

## Source repo investigated

Repository: `BradMoonUESTC/TxAnalyzer` cloned locally for inspection.

### Current stack

- Language: Python CLI/library.
- Runtime dependencies: `requests`, `pandas`, `typing-extensions`, `tqdm`, `web3`.
- EVM external tools: Foundry (`forge`, `anvil`, `cast`) and Heimdall for unverified EVM bytecode decompilation.
- Entry point: `scripts/pull_artifacts.py`.
- Core library: `txanalyzer/tx_analyzer.py`, `txanalyzer/transaction_processor.py`, `txanalyzer/heimdall_api.py`.
- Config: `config.json` copied from `config_template.json`, with network RPC and explorer API keys.
- Agent workflow: `SKILL.md` plus methodology docs in `docs/` drive the root-cause analysis after artifacts are pulled.

### Current artifact pipeline

For EVM networks, `pull_artifacts.py` does:

1. `trace_transaction` RPC.
2. Parse call tree and save transaction trace JSON / CSV.
3. Fetch contract source and ABI from Etherscan-compatible APIs.
4. Fetch bytecode and optionally decompile via Heimdall when source is unavailable.
5. `debug_traceTransaction` for opcode/struct logs.
6. Extract selectors and query `openchain.xyz` for function signatures.
7. Normalize into `transactions/<tx>/` with `trace/`, `contracts/`, `contract_sources/`, `opcode/`, and `README.md`.

The repo already contains a Solana path, which is useful as a design reference: it does **not** fake EVM-style opcode traces and instead stores standard-RPC transaction/meta payloads, instructions, logs, account snapshots, invoked program metadata/binaries, and capability notes.

### Key limitation to carry into Move

TxAnalyzer’s strongest EVM stages depend on EVM-only primitives:

- `trace_transaction`
- `debug_traceTransaction`
- opcode/stack/storage traces
- selector-to-signature lookup
- Etherscan-style verified source retrieval
- Foundry/anvil exact transaction-prestate fork replay

Sui and Aptos do not expose direct equivalents for all of these. The Move version should be evidence-complete for each chain, not an EVM abstraction layer with missing fields.

## Recommended CLI shape

Proposed package name: `move-txanalyzer`.

```bash
move-txanalyzer pull --network sui --tx <digest>
move-txanalyzer pull --network aptos --tx <hash-or-version>
move-txanalyzer analyze --network sui --tx <digest>
move-txanalyzer analyze --network aptos --tx <hash-or-version>
move-txanalyzer benchmark --cases benchmarks/move_cases.yaml --model <agent/model>
```

Output layout:

```text
transactions/<network>/<tx>/
├── manifest.json
├── trace/
│   ├── transaction.json
│   ├── effects_or_changes.json
│   ├── events.json
│   ├── balance_diffs.json
│   ├── calls_or_commands.json
│   └── summary.json
├── state/
│   ├── pre/                 # where the chain supports versioned reads
│   └── post/
├── packages/
│   └── <package-or-address>/
│       ├── metadata.json
│       ├── normalized_modules.json
│       ├── source_status.json
│       └── source/           # best-effort public source, if resolved
├── analysis/
│   └── result.md
└── capabilities.json
```

## Sui migration design

### Evidence to pull

Use Sui transaction-block and object APIs as the first-class artifact source:

- transaction block by digest with options for input, effects, events, object changes, balance changes, and raw input
- checkpoint / timestamp / gas / status
- Programmable Transaction Block command list
- changed object refs and owner changes
- created/mutated/deleted/wrapped/published object changes
- balance changes by owner/coin type
- events emitted by package/module
- package metadata and normalized Move modules/functions/structs
- object snapshots for important objects; when available, versioned object reads for before/after reconstruction

### Sui analysis model

Do not reason in EVM calls. Reason in:

`PTB command -> Move call/package -> object privilege/ownership -> object mutation -> balance/object ownership effect -> profit/control impact`

Important Sui-specific questions:

- Which shared objects were touched, and were they used as intended?
- Which owned objects / capabilities / coins moved owner?
- Did the PTB composition itself create the exploit path?
- Did a package publish/upgrade occur?
- Were dynamic fields, kiosks, transfer policies, or versioned packages involved?
- Was the victim loss visible as balance changes, object transfers, or internal accounting mutation?

### Sui modules to reuse from this repo

The existing `move-skills` repository already has Sui audit references that map well into transaction RCA, especially:

- `object-ownership.md`
- `ptb-composability.md`
- `token-flow-tracing.md`
- `dynamic-field` / object and transfer-policy references where applicable
- `package-version-safety.md`
- `oracle-analysis.md`
- `flash-loan-interaction.md`

## Aptos migration design

### Evidence to pull

Use Aptos transaction and account/module APIs as the first-class artifact source:

- transaction by hash or version
- payload, events, gas, status, VM status
- write set / changes, including resource writes/deletes and table item writes/deletes
- user transaction sender, sequence number, entry function, type args, args
- affected account resources at ledger versions where possible
- published modules / package metadata
- normalized Move modules/functions/structs
- events by handle where follow-up context is needed

### Aptos analysis model

Aptos should follow:

`entry function -> signer/resource authority -> resource/table write -> event/balance diff -> downstream acceptance -> profit/control impact`

Important Aptos-specific questions:

- Which signer authorized the entry function?
- Which account resources changed, and at what ledger version?
- Were capabilities (`SignerCapability`, `MintRef`, `BurnRef`, `TransferRef`, etc.) created, stored, leaked, or reused?
- Were Fungible Asset / Coin resources or stores mutated consistently?
- Were table/smart-vector writes economically meaningful?
- Did Move 2.2 dynamic dispatch / FA hooks / reentrancy-like patterns matter?

### Aptos modules to reuse from this repo

The existing Aptos references are directly useful for RCA triage:

- `token-flow-tracing.md`
- `fungible-asset-security.md`
- `ref-lifecycle.md`
- `reentrancy-analysis.md`
- `ability-analysis.md`
- `external-precondition-audit.md`
- `oracle-analysis.md`
- `migration-analysis.md`

## Suggested implementation plan

### Phase 1 — artifact pullers

Implement a Python CLI with chain adapters:

```text
move_txanalyzer/
├── cli.py
├── config.py
├── adapters/
│   ├── base.py
│   ├── sui.py
│   └── aptos.py
├── artifacts/
│   ├── writer.py
│   └── schemas.py
├── analysis/
│   ├── summarizer.py
│   ├── sui_methodology.md
│   └── aptos_methodology.md
└── benchmarks/
```

Adapter interface:

```python
class ChainAdapter:
    def pull_transaction(self, tx_id: str) -> TransactionArtifact: ...
    def pull_related_state(self, artifact: TransactionArtifact) -> StateBundle: ...
    def pull_package_metadata(self, artifact: TransactionArtifact) -> PackageBundle: ...
    def capabilities(self) -> dict: ...
```

### Phase 2 — normalized artifact schema

Keep raw chain artifacts, but add normalized summaries:

- participants: signers, senders, gas payer, owners, affected accounts/objects
- execution units: Sui PTB commands or Aptos entry/changes
- value movement: coin/SPL-like balance deltas, object/resource ownership changes
- control movement: capability/resource/object authority changes
- packages/modules touched
- unresolved evidence gaps

### Phase 3 — agent methodology

Create two methodology docs rather than one generic Move doc:

- Sui: PTB/object/shared-object/owner/capability oriented.
- Aptos: signer/resource/write-set/table/capability oriented.

Then integrate existing `move-skills` audit modules as triggered checklists.

### Phase 4 — optional source matching

Explorer/source support should be best-effort, not a hard dependency:

- Sui: package ID -> normalized modules; attempt public source lookup through protocol repo metadata/manual config.
- Aptos: address/module -> normalized module; attempt source lookup through published package metadata/protocol repo/manual config.
- If exact deployed source cannot be matched, mark source attribution as partial and keep the RCA anchored to on-chain artifacts.

### Phase 5 — replay/simulation layer

Do not promise an EVM-style transaction-prestate replay initially.

Initial safe approach:

- Sui: use dry-run/dev-inspect only for controlled reproduction hypotheses where inputs can be reconstructed; otherwise produce evidence-only RCA.
- Aptos: use view/simulate transaction where feasible; otherwise rely on write-set and ledger-version resource diffs.
- Mark replay confidence separately from evidence confidence.

## Benchmarks to run

### Benchmark dimensions

1. **Artifact completeness**
   - Sui: transaction/effects/events/object changes/balance changes/package metadata coverage.
   - Aptos: transaction/payload/events/write-set/resource/table/module coverage.

2. **RCA accuracy**
   - Compare `analysis/result.md` against human incident reports.
   - Label: match / partial / mismatch.

3. **Evidence grounding**
   - Count claims backed by raw artifact paths.
   - Penalize unsupported source-level claims when exact source is unavailable.

4. **False exploit rate**
   - Include benign maintenance / normal app transactions and ensure the tool does not force exploit narratives.

5. **Time/cost**
   - Time to pull artifacts.
   - Token cost / model runtime for analysis.

6. **Replay/simulation success**
   - Separately measure whether a dry-run/simulation/reproduction is possible, not whether the RCA is correct.

### Suggested benchmark set

Start with 20–30 cases:

- 8–10 Sui confirmed incidents or public postmortems.
- 8–10 Aptos confirmed incidents or public postmortems.
- 4–6 benign but complex transactions (DEX swaps, staking, governance, package publish/upgrade).
- 4–6 adversarial/ambiguous cases where source is unavailable or only partial.

Each benchmark row should contain:

```yaml
- id: sui-example-001
  network: sui
  tx: "<digest>"
  label: confirmed_exploit
  protocol: "..."
  human_report_url: "..."
  expected_root_cause: "short canonical root cause"
  expected_loss_assets: ["..."]
  required_evidence:
    - "object/balance/resource changed"
    - "authority/capability path identified"
  notes: "source available / source unavailable / replay possible"
```

## Major risks

- Exact source matching is weaker on Sui/Aptos than Etherscan-style source retrieval.
- Sui object versions and Aptos ledger versions must be handled carefully or state diffs will be misleading.
- PTB/CPI-like composition can make root cause appear in a caller/aggregator rather than the package that directly moved assets.
- Move vulnerabilities are often authority/object/resource invariant failures, not opcode-level bugs; the methodology must stay Move-native.
- Replay will likely be partial at first; benchmark it separately from RCA quality.

## Recommended next step

Implement Phase 1 artifact pullers and a small benchmark harness first. The useful MVP is not a perfect replay engine; it is a reliable evidence bundle plus a Move-native RCA methodology that can say exactly what changed, who had authority, and where the value/control moved.
