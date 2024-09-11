---
sponsor: "Basin"
slug: "2024-08-basin"
date: "2024-09-11"
title: "Basin Invitational"
findings: "https://github.com/code-423n4/2024-08-basin-findings/issues"
contest: 430
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Basin smart contract system written in Solidity. The audit took place between August 23 — August 27, 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 4 wardens participated:

  1. [Egis\_Security](https://code4rena.com/@Egis_Security) ([deth](https://code4rena.com/@deth) and [nmirchev8](https://code4rena.com/@nmirchev8))
  2. [ZanyBonzy](https://code4rena.com/@ZanyBonzy)
  3. [zanderbyte](https://code4rena.com/@zanderbyte)

This audit was judged by [0xsomeone](https://code4rena.com/@0xsomeone).

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 1 unique vulnerability in the category of MEDIUM severity.

Additionally, C4 analysis included 2 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Basin repository](https://github.com/code-423n4/2024-08-basin), and is composed of 3 smart contracts written in the Solidity programming language and includes 2433 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (1)
## [[M-01] `Stable2::calcLpTokenSupply()` function cannot convert under certain circumstances, DoSing `calcReserveAtRatioLiquidity`](https://github.com/code-423n4/2024-08-basin-findings/issues/5)
*Submitted by [Egis\_Security](https://github.com/code-423n4/2024-08-basin-findings/issues/5)*

`calcLpTokenSupply` is used in both `calcReserveAtRatioLiquidity` and `calcReserveAtRatioSwap`, we'll focus on `calcReserveAtRatioLiquidity`.

`calcLpTokenSupply` is called inside `calcRate` here:

```solidity
function calcReserveAtRatioLiquidity(
        uint256[] calldata reserves,
        uint256 j,
        uint256[] calldata ratios,
        bytes calldata data
    ) external view returns (uint256 reserve) {
...
for (uint256 k; k < 255; k++) {
            scaledReserves[j] = updateReserve(pd, scaledReserves[j]);
            // calculate new price from reserves:
            pd.newPrice = calcRate(scaledReserves, i, j, abi.encode(18, 18));
...
```

Inside we call the function:

```solidity
function calcRate(
        uint256[] memory reserves,
        uint256 i,
        uint256 j,
        bytes memory data
    ) public view returns (uint256 rate) {
        uint256[] memory decimals = decodeWellData(data);
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

        // calc lp token supply (note: `scaledReserves` is scaled up, and does not require bytes).
        uint256 lpTokenSupply = calcLpTokenSupply(scaledReserves, abi.encode(18, 18));
        rate = _calcRate(scaledReserves, i, j, lpTokenSupply);
    }
```

Note that the only change to `calcLpTokenSupply` is the added revert on the last line of the function.

```solidity
function calcLpTokenSupply(
        uint256[] memory reserves,
        bytes memory data
    ) public view returns (uint256 lpTokenSupply) {
        if (reserves[0] == 0 && reserves[1] == 0) return 0;
        uint256[] memory decimals = decodeWellData(data);
        // scale reserves to 18 decimals.
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

        uint256 Ann = a * N * N;
        
        uint256 sumReserves = scaledReserves[0] + scaledReserves[1];
        lpTokenSupply = sumReserves;
        for (uint256 i = 0; i < 255; i++) {
            uint256 dP = lpTokenSupply;
            // If division by 0, this will be borked: only withdrawal will work. And that is good
            dP = dP * lpTokenSupply / (scaledReserves[0] * N);
            dP = dP * lpTokenSupply / (scaledReserves[1] * N);
            uint256 prevReserves = lpTokenSupply;
            lpTokenSupply = (Ann * sumReserves / A_PRECISION + (dP * N)) * lpTokenSupply
                / (((Ann - A_PRECISION) * lpTokenSupply / A_PRECISION) + ((N + 1) * dP));
            // Equality with the precision of 1
            if (lpTokenSupply > prevReserves) {
                if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
            } else {
                if (prevReserves - lpTokenSupply <= 1) return lpTokenSupply;
            }
        }
        revert("Non convergence: calcLpTokenSupply");
    }
```

The issue here lies that `calcLpTokenSupply` reverts under certain circumstances since it cannot converge, which makes the whole call to `calcReserveAtRatioLiquidity` revert. We'll investigate this further inside the PoC section.

### Proof of Concept

Paste the following inside `BeanstalkStable2LiquidityTest` and run `forge test --mt test_calcReserveAtRatioLiquidity_equal_equalPositive -vv`. This is a rough test to loop through some of the cases in `Stable2LUT1`.

```solidity
function test_calcReserveAtRatioLiquidity_equal_equalPositive() public view {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1987e18;
        reserves[1] = 12345e8;
        uint256[] memory ratios = new uint256[](2);
        ratios[0] = 1e18;
        ratios[1] = 1e18;

        for(uint i; i < 10000; i++) {
            _f.calcReserveAtRatioLiquidity(reserves, 0, ratios, data);
            ratios[0] += 0.003e18;
        }
    }
```

You'll notice that the test reverts with `[FAIL. Reason: revert: Non convergence: calcLpTokenSupply]`.

The values that it reverts with are:

```solidity
  Ratios[0]:      7195000000000000000 
  Ratios[1]:      1000000000000000000
  Target price:   138927
  High Price:     277020
  Low Price:      1083
  Current price:  1083 (price is closer to lowPrice)
```

This is when we hit this case in `Stable2LUT1`:

```solidity
 if (price < 0.001083e6) {
                                    revert("LUT: Invalid price");
                                } else {
                                    return
             ->                         PriceData(0.27702e6, 0, 9.646293093274934449e18, 0.001083e6, 0, 2000e18, 1e18);
                                }
```

Here are the last few iterations of `calcLpTokenSupply`:

```solidity
 Current Lp token supply:  237151473819804594336
  Prev Lp token supply:     237151473819804594338
  -------------------------------------------
  Current Lp token supply:  237151473819804594338
  Prev Lp token supply:     237151473819804594336
  -------------------------------------------
  Current Lp token supply:  237151473819804594336
  Prev Lp token supply:     237151473819804594338
  -------------------------------------------
  Current Lp token supply:  237151473819804594338
  Prev Lp token supply:     237151473819804594336
  -------------------------------------------
  Current Lp token supply:  237151473819804594336
  Prev Lp token supply:     237151473819804594338
  -------------------------------------------
  Current Lp token supply:  237151473819804594338
  Prev Lp token supply:     237151473819804594336
  -------------------------------------------
```

As you can see, the values "loop" `+-2`; which makes it impossible to converge since the difference has to be `<= 1` so it converges.

Something to note is that even if we change the `reserves` in the test, the test continues failing with the same error, but every test fails when we hit this specific case in the lookup table and the price is closer to the `lowPrice`.

`PriceData(0.27702e6, 0, 9.646293093274934449e18, 0.001083e6, 0, 2000e18, 1e18);`

It's unclear what causes this "looping" to occur, so it's hard to say what the root cause is.

Something interesting is that this started happening once the revert was introduced, the last audit didn't have it, but the code still worked. Then it didn't revert it just returned the last `lpTokenSupply`.

Also note that the underlying issue is in `calcLpTokenSupply`, so if the function is used on its own it can still revert, but it happens much more often when `calcReserveAtRatioLiquidity` is used.

### Tools Used

Foundry

### Recommended Mitigation Steps

It's very hard to recommend a fix here, as many tweaks can fix the issue. We recommend:

1. Passing a flag to `calcLpTokenSupply`, if true then if the function doesn't converge it won't revert.
2. Tweaking the `PriceData` values for the specific case in the lookup table, narrowing down the range seems to fix the issue.

### Assessed type

DoS

**[Brean0 (Basin) confirmed](https://github.com/code-423n4/2024-08-basin-findings/issues/5#event-14057868666)**

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-08-basin-findings/issues/5#issuecomment-2324937113):**
 > The Warden has outlined how the system might fail to converge under normal `PriceData` configurations due to looping between the threshold by a deviancy of `±2`. I believe the vulnerability is valid as it causes normal operation of the system to result in a revert due to remediations carried out for a submission in the previous audit of Basin.
> 
> To note, a percentage-based deviation threshold might be better appropriate than a fixed unit (i.e., a permitted deviancy of `1`) to avoid instances whereby the permitted deviation itself is missed by a negligible amount.

***

# Low Risk and Non-Critical Issues

For this audit, 2 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-08-basin-findings/issues/6) by **Egis_Security** received the top score from the judge.

*The following wardens also submitted reports: [ZanyBonzy](https://github.com/code-423n4/2024-08-basin-findings/issues/3).*

## [01] `___self` param introduced by `WellUpgradeable` is redundant

[`WellUpgradeable`](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L16-L17) inherits from `UUPSUpgradeable`, which implements an immutable variable [`__self`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/22489db15621b9a42ebddb1facade6962034e9b9/contracts/proxy/utils/UUPSUpgradeable.sol#L22): `address private immutable __self = address(this);`

It is set to the address of the deployed implementation contract because it an `immutable` var and it is stored in the bytecode. The Basin team implements corresponding `___self` param, which is assigned similarly: `address  private  immutable ___self =  address(this);`

We can summarize that `__self == ___self`. However, it is impossible for `WellUpgradeable` to use `__self` variable, because it is marked as `private`. 
    
### Recommended Mitigation Steps

Duplicate `UUPSUpgradeable` and modify `__self` to be marked as `internal`, so you can reuse it inside `WellUpgradeable` and safe gas for deploying same argument two times.

## [02] `calcLpTokenSupply` can't converge for large reserve ratios, or very small reserves
  
[`calcLpTokenSupply`](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L75-L76) is used to calculate the `lpTokenSupply` for given reserves. We found some values for reserves (with large ratio difference), or very small reserves, which result in impossibility to converge and repeatedly calculating the same `lpTokenSupply`. 

**Issue 1:**

For large ratio reserves - one such `reserves` are:

```
Reserves 0 = 99999999000000000001246507 
Reserves 1 = 1000000000000000000
```

**Issue 2:**

For very small reserve amounts the same issue also arises - it only happens with the new code:

```
// update scaledReserve[j] such that calcRate(scaledReserves, i, j) = low/high Price,

// depending on which is closer to targetPrice.

if (pd.lutData.highPrice - pd.targetPrice > pd.targetPrice - pd.lutData.lowPrice) {
// targetPrice is closer to lowPrice.
scaledReserves[j] = scaledReserves[i] * pd.lutData.lowPriceJ / pd.lutData.precision;
// set current price to lowPrice.
pd.currentPrice = pd.lutData.lowPrice;
} else {
// targetPrice is closer to highPrice.
scaledReserves[j] = scaledReserves[i] * pd.lutData.highPriceJ / pd.lutData.precision;
// set current price to highPrice.
pd.currentPrice = pd.lutData.highPrice;

}
// calculate max step size:
// lowPriceJ will always be larger than highPriceJ so a check here is unnecessary.
pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
```

### Proof of Concept

Testcase for **Issue 2**:

```
function setUp() public {
        address lut = address(new Stable2LUT1());
        _f = new Stable2(lut);
        deployMockTokens(2);
        data = abi.encode(18, 18);
    }

function test_calcReserveAtRatioLiquidity_smallReserveRevert() public view {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1e7;
        reserves[1] = 1e7;
        uint256[] memory ratios = new uint256[](2);
        ratios[0] = 1027000000000000000;
        ratios[1] = 1000000000000000000;

        _f.calcReserveAtRatioLiquidity(reserves, 0, ratios, data);
    }
```

### Recommended Mitigation Steps

We didn't manage to identify a solution to this problem, but the behaviour can be documented with a `@dev` tag.

## [03] Potential precision and stability issues with equal values for amplification parameter

In `Stable2` we have an amplification param `a` and amplification precision scaling parameter [A_PRECISION](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L38-L39). [a](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/StableLUT/Stable2LUT1.sol#L19-L22) is supposed to be scaled by `A_PRECISION` to maintain precision in calculations, but currently, both values are set to `100`. The following results in having amplification factor equal to 1 and will result in an invariant, which looks more like uniswap constant product formula and swaps may be more exposed to slippage.

### Proof of Concept

The formula will have the following graph:
- **Yellow** is the formula in Basin Stable implementation
- **Pink** is Uniswap constant product formula
- **White** is Constant sum formula

*Note: to view the provided image, please see the original submission [here](https://github.com/code-423n4/2024-08-basin-findings/blob/main/data/Egis_Security-Q.md#qa-04-potential-precision-and-stability-issues-with-equal-values-for-amplification-parameter).*

### Recommended Mitigation Steps

Make `A` coefficient larger, or introduce a setter to dynamically adjust the value.

## [04] `WellUpgraedable#upgradeTo` consider renaming `newImplementation` to `newMinimalProxyImplementation`

In [`WellUpgraedable#upgradeTo`](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L99-L114) and `upgradeToAndCall` address argument, which is passed is named `newImplementation`, but in Basin's case, it should be a minimal proxy address, which has been deployed through the `Aquifier`.

### Recommended Mitigation Steps

Consider renaming `newImplementation` to `newMinimalProxyImplementation` to make the code cleaner for potential integrators.

**[Brean0 (Basin) commented](https://github.com/code-423n4/2024-08-basin-findings/issues/6#issuecomment-2317121461):**
 > [01] - While the variable is indeed redundant, modifying `__self` on the `UUPSUpgradeable` contract would require us to have a custom implementation of the UUPSUpgradeable contract. The Basin Development Community decided this was preferable at the marginal cost of gas. 
 >
> [02] - Accepted, the docs will be updated to reflect that the Well Function cannot be used for extremely high or low reserves. 
> 
> [03] - This is a design choice and is intentional. Developers can choose an `A` parameter that suits the need of the protocol by deploying a lookup table with the desired `A` parameter.  
> 
> [04] - Accepted, the parameter will be updated so that it's clearer for developers that the implementation should be the minimal proxy deployed by an Aquifer. 

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
