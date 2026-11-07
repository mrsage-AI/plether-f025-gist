# F-025 — PoC

forge test --match-contract F025SameSecondMarkStallTest -vv (tmp-f025, pinned a82d4fe, forge 1.9.0, solc 0.8.35, via_ir): 3/3 PASS.
- A: same-second moved-price update reverts PletherOracle__EqualTimeMarkMismatch (0x5e2d85fd) on updateLiquidationPrice AND updateLiquidationBatchPrice; batch pays the 1e18 wei Pyth fee then reverts wholesale before any per-account liquidation (185,187 gas burned, fee rolled back atomically).
- B: the identical shape at second T+1 succeeds — mark refreshed to P2@T+1, position liquidated.
- C: empty-pythUpdateData liquidation during the stall executes at the stale accepted price P1 with zero fee — degradation, not full stall.
Files: packages/perps/test/F025SameSecondMarkStall.t.sol (repo files unmodified). Full source:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {LiquidationBatchTestFixture} from "perps-test/perps/shared/LiquidationBatchTestFixture.sol";

/// @notice PoC for F-025 (MED-18 fix-introduced same-second mark-stall), pinned at a82d4fe.
/// @dev Frozen assertions:
///      A. Same-second oracle update with moved price reverts PletherOracle__EqualTimeMarkMismatch on
///         updateLiquidationPrice AND updateLiquidationBatchPrice (batch reverts wholesale after the fee quote).
///      B. The identical shape at second T+1 succeeds — the stall is strictly the same-second window.
///      C. A liquidation with EMPTY pythUpdateData during the stall succeeds at the stale accepted price
///         P1 with zero fee — liquidations degrade, mark refresh is hard-stalled.
///      Measure: gas burned by the reverted batch attempt and the Pyth fee quoted/forwarded.
contract F025SameSecondMarkStallTest is LiquidationBatchTestFixture {
    uint256 internal constant P_OPEN = 100_000_000; // $1.00, entry mark
    uint256 internal constant P_STALE = 101_500_000; // P1: accepted live basket at second T
    uint256 internal constant P_MOVED = 102_000_000; // P2 != P1, all components still at second T
    uint256 internal constant T0 = 1_729_080_000; // Wednesday noon UTC, outside FAD/frozen windows
    uint256 internal constant FEE = 1 ether;

    address internal constant LIQ = address(0xBA7C0F25);

    uint64 internal stallTime;

    function setUp() public override {
        super.setUp();
        baseMockPyth.setFee(FEE);
        _setupStallState();
    }

    /// @dev Accepted live basket at second T with price P1 (the MED-18 fix's state) and a long
    ///      position that is liquidatable at P1.
    function _setupStallState() internal {
        vm.roll(block.number + 1);
        vm.warp(T0);
        router.updateMarkPrice{value: FEE}(_payloadAt(P_OPEN)); // accepted basket 1.00 @ T0, engine mark 1.00 @ T0
        _fundTrader(LIQ, 2100e6);
        _open(LIQ, CfdTypes.Side.LONG, 100_000e18, 2000e6, P_OPEN);

        vm.roll(block.number + 1);
        vm.warp(T0 + 1);
        stallTime = uint64(vm.getBlockTimestamp());
        router.updateMarkPrice{value: FEE}(_payloadAt(P_STALE)); // accepted basket P1 @ T, engine mark P1 @ T
        assertTrue(
            engineLens.previewLiquidation(LIQ, P_STALE).liquidatable,
            "setup: account must be liquidatable at the stale accepted price"
        );
        assertEq(engine.lastMarkPrice(), P_STALE, "setup: mark must equal accepted basket price");
        assertEq(uint256(engine.lastMarkTime()), stallTime, "setup: mark time must equal second T");
    }

    function _payloadAt(uint256 price) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _decodeMismatch(bytes memory ret)
        internal
        pure
        returns (uint64 publishTime, uint256 basketPrice, uint256 enginePrice)
    {
        bytes memory tail = new bytes(ret.length - 4);
        for (uint256 i; i < tail.length; ++i) {
            tail[i] = ret[i + 4];
        }
        (publishTime, basketPrice, enginePrice) = abi.decode(tail, (uint64, uint256, uint256));
    }

    function _oneAccount() internal view returns (address[] memory accounts) {
        accounts = new address[](1);
        accounts[0] = LIQ;
    }

    /// ASSERTION A — the stall: same-second moved-price update reverts both liquidation oracle paths.
    function test_F025_A_SameSecondMarkStall() public {
        bytes[] memory moved = _payloadAt(P_MOVED); // all components publish at second T

        // Single-account path: oracle.updateLiquidationPrice
        vm.deal(KEEPER, 10 ether);
        vm.prank(KEEPER);
        (bool okSingle, bytes memory retSingle) =
            address(router).call{value: FEE}(abi.encodeCall(router.executeLiquidation, (LIQ, moved)));
        assertFalse(okSingle, "A: single liquidation must revert in the same-second stall");
        assertTrue(
            bytes4(retSingle) == IPletherOracle.PletherOracle__EqualTimeMarkMismatch.selector,
            "A: single path revert selector"
        );
        (uint64 ptSingle, uint256 basketSingle, uint256 engineSingle) = _decodeMismatch(retSingle);
        assertEq(uint256(ptSingle), stallTime, "A: revert publishTime is second T");
        assertEq(basketSingle, P_MOVED, "A: revert basket price is the moved price");
        assertEq(engineSingle, P_STALE, "A: revert accepted price is P1");

        // Batch path: pays the Pyth fee quote then reverts wholesale.
        assertEq(pletherOracle.getUpdateFee(moved), FEE, "A: fee quoted for moved update");
        uint256 parseBefore = baseMockPyth.parseCallCount();
        uint256 keeperEthBefore = KEEPER.balance;
        uint256 pythEthBefore = address(baseMockPyth).balance;
        uint256 g0 = gasleft();
        vm.prank(KEEPER);
        (bool okBatch, bytes memory retBatch) = address(router).call{value: FEE}(
            abi.encodeCall(IPerpsKeeper.executeLiquidationBatch, (_oneAccount(), moved))
        );
        uint256 batchRevertGas = g0 - gasleft();
        assertFalse(okBatch, "A: batch liquidation must revert wholesale");
        assertTrue(
            bytes4(retBatch) == IPletherOracle.PletherOracle__EqualTimeMarkMismatch.selector,
            "A: batch path revert selector"
        );
        assertEq(KEEPER.balance, keeperEthBefore, "A: fee payment rolled back atomically");
        assertEq(address(baseMockPyth).balance, pythEthBefore, "A: Pyth retains no fee after rollback");
        assertEq(baseMockPyth.parseCallCount(), parseBefore, "A: parse frame rolled back wholesale");
        assertEq(engine.lastMarkPrice(), P_STALE, "A: mark refresh hard-stalled at P1");
        assertEq(uint256(engine.lastMarkTime()), stallTime, "A: mark time still second T");

        console2.log("F025 batch-revert gas burned:", batchRevertGas);
        console2.log("F025 pyth fee quoted/forwarded (wei):", FEE);
    }

    /// ASSERTION B — impact boundary: the same shape at T+1 succeeds.
    function test_F025_B_NextSecondRecovers() public {
        vm.deal(KEEPER, 10 ether);
        bytes[] memory movedNow = _payloadAt(P_MOVED);
        vm.prank(KEEPER);
        (bool okNow, bytes memory retNow) = address(router).call{value: FEE}(
            abi.encodeCall(IPerpsKeeper.executeLiquidationBatch, (_oneAccount(), movedNow))
        );
        assertFalse(okNow, "B: same-second attempt must still revert");
        assertTrue(bytes4(retNow) == IPletherOracle.PletherOracle__EqualTimeMarkMismatch.selector, "B: stall selector");

        vm.roll(block.number + 1);
        vm.warp(stallTime + 1);
        bytes[] memory movedNext = _payloadAt(P_MOVED); // all components at T+1
        vm.prank(KEEPER);
        uint256 nextIndex =
            IPerpsKeeper(address(router)).executeLiquidationBatch{value: FEE}(_oneAccount(), movedNext);
        assertEq(nextIndex, 1, "B: batch must complete at T+1");
        assertEq(_positionSize(LIQ), 0, "B: position liquidated at T+1");
        assertEq(engine.lastMarkPrice(), P_MOVED, "B: mark refreshed to the moved price");
        assertEq(uint256(engine.lastMarkTime()), stallTime + 1, "B: mark time advanced to T+1");
    }

    /// ASSERTION C — degradation: empty-pythUpdateData liquidation succeeds at the stale price with zero fee.
    function test_F025_C_EmptyDataLiquidationDegradesAtStalePrice() public {
        bytes[] memory empty = new bytes[](0);
        uint256 parseBefore = baseMockPyth.parseCallCount();
        uint256 keeperEthBefore = KEEPER.balance;

        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeLiquidation(LIQ, empty);
        Vm.Log[] memory logs = vm.getRecordedLogs();

        (bool found, CfdTypes.Side side, uint256 execPrice) = _liquidationEvent(logs, LIQ);
        assertTrue(found, "C: liquidation must execute during the stall");
        assertEq(uint256(side), uint256(CfdTypes.Side.LONG), "C: side");
        assertEq(execPrice, P_STALE, "C: executed at the stale accepted price P1");
        assertEq(_positionSize(LIQ), 0, "C: position wiped at the stale price");
        assertEq(engine.lastMarkPrice(), P_STALE, "C: mark REFRESH stays hard-stalled at P1");
        assertEq(uint256(engine.lastMarkTime()), stallTime, "C: mark time stays second T");
        assertEq(baseMockPyth.parseCallCount(), parseBefore, "C: no Pyth parse for empty update data");
        assertEq(KEEPER.balance, keeperEthBefore, "C: zero fee for empty update data");
    }
}
```

Revert site: PletherOracle._resolveCurrentBasket :793, before the router's engine.updateMarkPrice.


## Runnable test source

Toolchain: forge 1.9.0 (the program-stated Foundry 1.5.1 builds the same tree; repo foundry.toml pins solc 0.8.35, via_ir), 3/3 PASS on pinned a82d4fe.

`packages/perps/test/F025SameSecondMarkStall.t.sol` — full source reproduced in the PoC proof record (seat reply 289ea3dc, archived at prove/poc-F-025/); imports `LiquidationBatchTestFixture` + `MockPyth` from the repo's own test-support, and asserts:
- A: same-second moved-price update reverts `PletherOracle__EqualTimeMarkMismatch` (0x5e2d85fd) on `updateLiquidationPrice` AND `updateLiquidationBatchPrice`; batch pays the 1e18 wei Pyth fee then reverts wholesale before any per-account liquidation (185,187 gas burned, fee rolled back atomically).
- B: the identical shape at second T+1 succeeds — mark refreshed to P2@T+1, position liquidated.
- C: empty-pythUpdateData liquidation during the stall executes at the stale accepted price P1 with zero fee.
