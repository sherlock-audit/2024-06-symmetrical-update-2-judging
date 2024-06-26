Dancing Navy Condor

Medium

# Collateral can still be allocated to PartyA when the system is paused by exploiting the new internal transfer function

## Summary

Collateral can still be allocated to PartyA when the system is paused by exploiting the new internal transfer function.

## Vulnerability Detail

The `allocate` and `depositAndAllocate` functions are guarded by the `whenNotAccountingPaused` modifier to ensure that collateral can only be allocated when the accounting is not paused.

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L48

```solidity
File: AccountFacet.sol
46: 	/// @notice Allows Party A to allocate a specified amount of collateral. Allocated amounts are which user can actually trade on.
47: 	/// @param amount The precise amount of collateral to be allocated, specified in 18 decimals.
48: 	function allocate(uint256 amount) external whenNotAccountingPaused notSuspended(msg.sender) notLiquidatedPartyA(msg.sender) {
49: 		AccountFacetImpl.allocate(amount);
..SNIP..
52: 	}
53: 
54: 	/// @notice Allows Party A to deposit a specified amount of collateral and immediately allocate it.
55: 	/// @param amount The precise amount of collateral to be deposited and allocated, specified in collateral decimals.
56: 	function depositAndAllocate(uint256 amount) external whenNotAccountingPaused notLiquidatedPartyA(msg.sender) notSuspended(msg.sender) {
57: 		AccountFacetImpl.deposit(msg.sender, amount);
58: 		uint256 amountWith18Decimals = (amount * 1e18) / (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
59: 		AccountFacetImpl.allocate(amountWith18Decimals);
..SNIP..
63: 	}
```

However, malicious users can bypass this restriction by exploiting the newly implemented `AccountFacet.internalTransfer` function. When the global pause (`globalPaused`) and accounting pause (`accountingPaused`) are enabled, malicious users can use the `AccountFacet.internalTransfer` function, which is not guarded by the `whenNotAccountingPaused` modifier, to continue allocating collateral to their accounts, effectively bypassing the pause.

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L79

```solidity
File: AccountFacet.sol
74: 	/// @notice Transfers the sender's deposited balance to the user allocated balance.
75: 	/// @dev The sender and the recipient user cannot be partyB.
76: 	/// @dev PartyA should not be in the liquidation process.
77: 	/// @param user The address of the user to whom the amount will be allocated.
78: 	/// @param amount The amount to transfer and allocate in 18 decimals.
79: 	function internalTransfer(address user, uint256 amount) external whenNotAccountingPaused whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
80: 		AccountFacetImpl.internalTransfer(user, amount);
..SNIP..
85: 	}

```

## Impact

When the global pause (`globalPaused`) and accounting pause (`accountingPaused`) are enabled, this might indicate that:

1) There is an issue, error, or bug in certain areas (e.g., accounting) of the system. Thus, the funds transfer should be halted to prevent further errors from accumulating and to prevent users from suffering further losses due to this issue
2) There is an ongoing attack in which the attack path involves transferring/allocating funds to an account. Thus, the global pause (`globalPaused`) and accounting pause (`accountingPaused`) have been activated to stop the attack. However, it does not work as intended, and the hackers can continue to exploit the system by leveraging the new internal transfer function to workaround the restriction.

In both scenarios, this could lead to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L79

## Tool used

Manual Review

## Recommendation

Add the `whenNotAccountingPaused` modifier to the `internalTransfer` function.

```diff
- function internalTransfer(address user, uint256 amount) external whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
+ function internalTransfer(address user, uint256 amount) external whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
  AccountFacetImpl.internalTransfer(user, amount);
```