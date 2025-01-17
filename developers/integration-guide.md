---
description: Find out how to start working with the Euler smart contracts
---

# Contract Integration Guide

## Modules

The Euler protocol is a collection of smart contracts, connected together with a module system. Each module handles specific areas of the protocol, so depending on what you want to do, you will interact with several different contract addresses.

Some modules are global, for example:

* [markets](integration-guide.md#markets): Activating markets, enter/exiting markets, and querying various market-related information.
* [exec](integration-guide.md#exec): Batch requests, liquidity deferrals \(ie, flash loans\)
* [liquidation](integration-guide.md#liquidation): Seizure of assets for users in violation

Other modules are asset-specific:

* [eTokens](integration-guide.md#eTokens): ERC20-compatible tokens that represent assets
* [dTokens](integration-guide.md#dTokens): ERC20-compatible tokens that represent liabilities

## Deposit and withdraw

In order to invest an asset to earn interest, you need to `deposit` into an eToken.

```javascript
// Approve the main euler contract to pull your tokens:
IERC20(underlying).approve(EULER_MAINNET, type(uint).max);

// Use the markets module:
IEulerMarkets markets = IEulerMarkets(EULER_MAINNET_MARKETS);

// Get the eToken address using the markets module:
IEulerEToken eToken = IEulerEToken(markets.underlyingToEToken(underlying));

// Deposit 5.25 underlying tokens (assuming 18 decimal places)
// The "0" argument refers to the sub-account you are depositing to.
eToken.deposit(0, 5.25e18);

eToken.balanceOf(address(this));
// -> internal book-keeping value that doesn't increase over time

eToken.balanceOfUnderlying(address(this));
// -> 5.25e18
// ... but check back next block to see it go up (assuming there are borrowers)

// Later on, withdraw your initial deposit and all earned interest:
eToken.withdraw(0, type(uint).max);
```

## Borrow and repay

If you would like to borrow an asset, you must have sufficient collateral, and be "entered" into the collateral's market.

```javascript
// Use the markets module:
IEulerMarkets markets = IEulerMarkets(EULER_MAINNET_MARKETS);

// Approve, get eToken addr, and deposit:
IERC20(collateral).approve(EULER_MAINNET, type(uint).max);
IEulerEToken collateralEToken = IEulerEToken(markets.underlyingToEToken(collateral));
collateralEToken.deposit(0, 100e18);

// Enter the collateral market (collateral's address, *not* the eToken address):
markets.enterMarket(0, collateral);

// Get the dToken address of the borrowed asset:
IEulerDToken borrowedDToken = IEulerDToken(markets.underlyingToDToken(borrowed));

// Borrow 2 tokens (assuming 18 decimal places).
// The 2 tokens will be sent to your wallet (ie, address(this)).
// This automatically enters you into the borrowed market.
borrowedDToken.borrow(0, 2e18);

borrowedDToken.balanceOf(address(this));
// -> 2e18
// ... but check back next block to see it go up

// Later on, to repay the 2 tokens plus interest:
IERC20(borrowed).approve(EULER_MAINNET, type(uint).max);
borrowedDToken.repay(0, type(uint).max);
```

## Flash loans

Euler doesn't have any specific flash loan functionality. Instead, you can defer the liquidity check for your account. The Euler contract will call back into your contract, where you can perform operations like `borrow()` without worrying about liquidity violations. As long as your callback leaves the account in a non-violating state, the transaction will complete successfully.

Since Euler only charges interest for a loan when it is held for a non-zero amount of time, this results in fee-less flash loans.

Here is an example contract that demonstrates this:

```javascript
contract MyFlashLoanContract {
    struct MyCallbackData {
        uint whatever;
    }

    function somethingThatNeedsFlashLoan() {
        // Setup whatever data you need
        MyCallbackData memory data;
        data.whatever = 1234;

        // Disable the liquidity check for "this" and call-back into onDeferredLiquidityCheck:
        IExec(exec).deferLiquidityCheck(address(this), abi.encode(data));
    }

    function onDeferredLiquidityCheck(bytes memory encodedData) external override {
        MyCallbackData memory data = abi.decode(encodedData, (MyCallbackData));

        // Borrow 10 tokens (assuming 18 decimals):

        IEulerDToken(borrowedDToken).borrow(0, 10e18);

        // ... do whatever you need with the borrowed tokens ...

        // Repay the 10 tokens:

        IERC20(borrowed).approve(EULER_MAINNET, type(uint).max);
        IEulerDToken(borrowedDToken).repay(0, 10e18);
    }
}
```

`encodedData` is a pass-through parameter that lets you transfer data to your callback without requiring storage writes.

