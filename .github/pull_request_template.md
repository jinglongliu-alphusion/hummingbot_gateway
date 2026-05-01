## Links

- Spec / Issue:
- Iteration ID:
- ADR:

## Change Summary

<!-- Purpose, connector/DEX scope, and user-visible behavior. -->

## 5-Axis Self Review

- [ ] Correctness: behavior matches spec; edge cases covered; no silent failure.
- [ ] Readability: names are clear; public APIs/configs are documented; comments explain why.
- [ ] Architecture: connector boundaries hold; no circular dependency; API contracts preserved.
- [ ] Security: no plaintext secrets; wallet/account permissions and RPC boundaries hold.
- [ ] Performance: no avoidable RPC latency or hot-path regression; p50/p99 impact assessed.

## DEX / Chain Checks

- [ ] Uniswap/Jupiter SDK calls do not introduce unexpected RPC latency or retry storms.
- [ ] Slippage, gas, quote freshness, and chain/account assumptions are documented if touched.

## AutoResearch Baseline Comparison

<!-- Required only for strategy/experiment-facing changes. Use N/A otherwise. -->

| Metric | Baseline | Proposal | Delta | Notes |
| --- | --- | --- | --- | --- |
| Sharpe | | | | |
| MaxDD | | | | |
| Latency p99 (us) | | | | |
| Win rate | | | | |

ExperimentTracker run id:

## Test Plan

- [ ] Unit tests:
- [ ] Integration / E2E:
- [ ] QA tips:

## New Dependencies

- [ ] No new dependencies.
- [ ] New dependencies listed with version, license, and purpose:

## Rollback

<!-- Revert path and operational rollback if needed. -->

## Checklist References

- strict delivery: `/opt/zquant/.codex/skills/zquant/strict-delivery/SKILL.md`
- AutoResearch: `/opt/zquant/.codex/skills/zquant/autoresearch/SKILL.md`
