# F-025 — gist

The MED-18 fix (equal-timestamp mark immutability) introduced a same-second stall: liquidation/mark-refresh payloads whose Pyth components all still sit at the accepted second revert EqualTimeMarkMismatch — batch attempts quote-and-forward the Pyth fee (rolled back atomically) while burning ~185k gas, mark refresh is hard-stalled until a strictly newer second, and liquidations degrade to the prior-second price. Fix-introduced-new-bug per the program's own eligibility rule.

---
**SUBMITTED on Immunefi: 08/11/26** (asset: packages/perps/src, severity Medium, impact Griefing). PoC independently re-run and confirmed by DeepSeek 1017 (tmp-f025c).
