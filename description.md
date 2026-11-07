# F-025 — MED-18 fix introduces a same-second mark-stall: fresh-price liquidation and mark-refresh payloads revert until a strictly newer oracle second

## Summary
The MED-18 remediation ("treat equal-timestamp marks as immutable by rejecting publishTime <= lastMarkTime in router/oracle mark-update paths", Cecuro ai-audit :1532, commits 2fca7941/7de2f742, per the Cecuro report) introduced a new liveness bug: any oracle update payload whose publish second is still T (the accepted basket's time — PletherOracle.sol:815-817 makes basket.publishTime the MIN component time) with a MOVED price now reverts `PletherOracle__EqualTimeMarkMismatch` on both liquidation entrypoints — `updateLiquidationPrice` and `updateLiquidationBatchPrice` (PletherOracle.sol:786-794 basket check, :896-901 engine-mark mirror, CfdEngine.sol:1088-1089). The batch path quotes and forwards the Pyth fee, then reverts wholesale before any per-account liquidation (185,187 gas measured; the fee is rolled back atomically but the keeper's gas is burned — repeatable every same-second repricing).

## Impact
- Mark REFRESH is hard-stalled during any same-second repricing window: the engine mark cannot advance to a moved price until Pyth publishes a strictly newer second (T+1). During fast markets this delays the mark that protection triggers, stop-losses and liquidation pricing depend on.
- Liquidations DEGRADE, not stall: with empty pythUpdateData the oracle serves the stale accepted basket (fee 0, PletherOracle.sol:461-466), so a keeper can still liquidate at the prior-second price — but fresh-price liquidation attempts (the ones that matter in a fast move) revert and burn gas until T+1.
- Category: fix-introduced-new-bug (explicitly eligible per program rules). MED-18's own remediation prescribes the arming behavior; neither the Cecuro report nor SECURITY.md:428 (which documents the invariant as a mitigation) acknowledges the liveness side effect.

## Measured
PROOF: SANDBOX tmp-f025 (ghcr.io/trycua/linux:24.04) PASS — equal-ts revert 0x5e2d85fd PletherOracle__EqualTimeMarkMismatch(uint64,uint256,uint256) on updateLiquidationPrice and updateLiquidationBatchPrice, T+1 identical shape succeeds (mark P2@T+1, position wiped), empty-data liquidation degrades at stale P1 with 0 fee, batch revert burned 185,187 gas with 1e18 wei fee quoted/forwarded-then-rolled-back

## Distinctness from known issues
- MED-18 (Addressed): the fix itself — we report its introduced bug, per the program's fix-incomplete/new-bug rule.
- Scan d4541b49 info/DoS "Later mark refresh can make live historical-price orders permanently out of order": order-execution/expiry path (bounty theft), not equal-timestamp liquidation-payload reverts or stale-price degradation.
- SECURITY.md:428: documents the equal-time invariant as a mitigation without the liveness side effect.
- SECURITY.md:687 "live basket component selection": about caller CHOICE of components, not the same-second stall the fix introduced.

## Severity
Medium (temporary denial of fresh-price liquidation/mark refresh; no direct fund loss; self-heals at T+1 but recurs on every fast-market same-second repricing).
