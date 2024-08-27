# QA for Basin Invitational
## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-self-param-introduced-by-wellupgradeable-is-redundant) | `___self` param introduced by `WellUpgradeable` is redundant |
| [QA-02](#qa-02-user-can-call-init-on-minimalproxy-address-of-wellupgradeable) | User can call `init` on minimalProxy address of `WellUpgradeable` |
| [QA-03](#qa-03-calclptokensupply-cant-converge-for-large-reserve-ratios) | `calcLpTokenSupply` can't converge for large reserve ratios |
| [QA-04](#qa-04-potential-precision-and-stability-issues-with-equal-values-for-amplification-parameter) | Potential precision and stability issues with equal values for amplification parameter |
| [QA-05](#qa-05-wellupgraedableupgradeto-consider-renaming-newimplementation-to-newminimalproxyimplementation) | `WellUpgraedable#upgradeTo` consider renaming `newImplementation` to `newMinimalProxyImplementation` |


## [QA-01] `___self` param introduced by `WellUpgradeable` is redundant

### Impact
[WellUpgradeable](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L16-L17) inherits from `UUPSUpgradeable`, which implements an immutable variable [__self](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/22489db15621b9a42ebddb1facade6962034e9b9/contracts/proxy/utils/UUPSUpgradeable.sol#L22): 
` address private immutable __self = address(this);`
It is set to the address of the deployed implementation contract, because it an `immutable` var and it is stored in the bytecode.
Basin team implements corresponding `___self` param, which is assigned similarly:
`address  private  immutable ___self =  address(this);`
So we can summerize that `__self == ___self`. 
However, it is impossible for `WellUpgradeable` to use `__self` variable, because it is marked as `private`. 
    
### Recommended Mitigation Steps
- Duplicate `UUPSUpgradeable` and modify `__self` to be marked as `internal`, so you can reuse it inside `WellUpgradeable` and safe gas for deploying same argument two times.


## [QA-02] User can call `init` on minimalProxy address of `WellUpgradeable`

### Impact
Current structure of project proxies is as follows:
 `UUPS Proxy` --(delegatecall)--> `MinimalProxy` --(delegatecall)--> `WellUpgradable Implementation`.
`WellUpgradable` has two `init` functions. 
`initNoWellToken` is called inside `Aquifier` when a  `MinimalProxy` is deployed. [init](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L31) is a `reinitializer(2)`, which is called on `UUPS Proxy` and set the owner in the storage of the calling context (`UUPS Proxy`). In this point anyone can call `init` on the `MinimalProxy` implementation, which will set the owner of the minimal proxy to the caller. 
Currently the impact is limited, because of the logic inside `_authorizeUpgrade` and `Aquifier`, but this may lead to potential future expoits, if `Aquifier` is changed and the following check may be manipulated:
```
address activeProxy =  IAquifer(aquifer).wellImplementation(_getImplementation());

require(activeProxy == ___self, "Function must be called through active proxy bored by an aquifer");
```
If a malicious owner manage to call `upgradeToAndCall` to the `MinimalProxy`, he will be able to `selfdestruct` the `MinimalProxy`, which is critical, because the original UUPS is using it's logic. 

### Proof of Concept
Example attacker contract exploiting `MiminalProxy#init` availability and trying to selfdestruct the `minimalproxy implementation`:
```
contract AttackerManager {

  constructor() {}
  function initiateAttack(address aquifer, address oldMinimalProxyImpl, IERC20[] memory tokens, address wellFunctionAddress) public {
    WellUpgradeable(oldMinimalProxyImpl).init("Test", "Test"); // We are now owners of the current minimal proxy implementation
    SelfDestructingImplementation newImpl = new SelfDestructingImplementation(); // Encode params do deploy the new contract 
    Call memory wellFunction = Call(wellFunctionAddress, abi.encode("2"));
    Call[] memory pumps = new Call[](1); (bytes memory immutableData, bytes memory initData) = LibWellUpgradeableConstructor.encodeWellDeploymentData(aquifer, tokens, wellFunction, pumps);
    // Bore the malicious contract in the Aquifier address newMinimalProxy = Aquifer(aquifer).boreWell(address(newImpl), immutableData, initData, bytes32(0)); 
    bytes memory encodedCalldata = abi.encodeWithSelector(SelfDestructingImplementation.selfDest.selector);
    WellUpgradeable(oldMinimalProxyImpl).upgradeToAndCall(newMinimalProxy, encodedCalldata);
  }
  // 1. Call `victim MinimalProxy.init, so we are the owners of it` 
  // 2. Deploy malicious implementation, which contains a selfdestructing function inside 
  // 3. Bore the malicious implementation inside `aquifier` 
  // 4. Call `victim MinimalProxy.upgradeToAndCall` with the minimalProxy, cloned from the malicious implementation and selfdestructing function 
}
contract SelfDestructingImplementation is WellUpgradeable {
  function selfDest() public {
    selfdestruct(payable(address(0)));
  }
}
```

### Recommended Mitigation Steps
Modify `_authorizeUpgrade` to always revert if `_getImplementation() == address(0)`

## [QA-03] `calcLpTokenSupply` can't converge for large reserve ratios, or very small reserves
 
 ### Impact
 
[calcLpTokenSupply](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L75-L76) is used to calculate the `lpTokenSupply` for given reserves. We found some values for reserves (with large ratio difference), or very small reserves, which result in impossibility to converge an repeatedly calculating same `lpTokenSupply`. 

- **Issue 1**  For large ratio reserves
One such `reserves` are:
```
Reserves 0 = 99999999000000000001246507 
Reserves 1 = 1000000000000000000
```

- **Issue, 2** For very small reserve amounts the same issue also arises
It only happens with the new code:
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
Testcase for issue #2
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

## [QA-04] Potential precision and stability issues with equal values for amplification parameter

### Impact
In `Stable2` we have an amplification param `a` and amplification precision scaling parameter [A_PRECISION](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L38-L39). [a](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/StableLUT/Stable2LUT1.sol#L19-L22) is supposed to be scaled by `A_PRECISION` to maintain precision in calculations, but currently both values are set to `100` The following results in having amplification factor equal to 1. 
The following will result in an invariant, which looks more like uniswap constant product formula and swaps may be more exposed to slippage.
### Proof of Concept
The formula will have the following grapth:
- **Yellow** is the formula in Basin Stable implementation
- **Pink** is Uniswap constant product formula
- **White** is Constant sum formula
![image](https://i.ibb.co/CbztNxv/Screenshot-2024-08-27-at-13-11-13.png)

### Recommended Mitigation Steps
Make A coefficient larger, or introduce a setter to dinamically adjust the value.

## [QA-05] `WellUpgraedable#upgradeTo` consider renaming `newImplementation` to `newMinimalProxyImplementation`

### Impact
In [WellUpgraedable#upgradeTo](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L99-L114) and `upgradeToAndCall` address argument, which is passed is named `newImplementation`, but in Basin's case it should be a minimal proxy address, which has been deployed through the `Aquifier`.

### Recommended Mitigation Steps

Consider renaming `newImplementation` to `newMinimalProxyImplementation` to make the code cleaner for potential integrators