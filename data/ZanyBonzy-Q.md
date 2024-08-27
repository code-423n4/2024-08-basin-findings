### 1. Protocol supports upgradeable tokens which can be upgraded to introduce unsupported token behaviours.

Links to affected code *

https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/README.md#erc20-token-behaviors-in-scope

#### Impact

According to the provided information in the readme, the protocol donly supports upgradable token types, no other weird tokens. The issue is that these tokens can introduce new behaviours in their new implementations. The token admins don't even have to be malicious, the new upgrade may just not be compatible with the protocol which may lead to serious unexpected issues. 
For instance, the token decimals may be changed, to be more or less than the initial decimals. In a case that the new decimals is more than 18, decoding the well data, will always fail dossing the protocol operations and if less than 18 or the initial, will affect the prices returned when functions in Stable2.sol are queried.

#### Recommended Mitigation Steps

It is possible to introduce logic that freezes interaction with the token in question if an upgrade is detected. An example of this is the [TUSD adapter ](https://github.com/makerdao/dss-deploy/blob/7394f6555daf5747686a1b29b2f46c6b2c64b061/src/join.sol#L321)used by MakerDAO.
***


***

### 2. `getRatiosFromPriceLiquidity()` and `getRatiosFromPriceSwap()` in Stable2LUT1.sol are not well optimized 

Links to affected code *

https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/StableLUT/Stable2LUT1.sol#L27-L2175

#### Impact

The `getRatiosFromPriceLiquidity()` and `getRatiosFromPriceSwap()` in Stable2LUT1.sol are not well optimized for readability or even gas as they contain repetitive nested if-else statements for handling different price ranges. This structure makes the code difficult to read, maintain, and scale as the number of price ranges increases. This method also makes a lot of checks extremely redundant as two price conditions may be satisfied, but only one will be in use. For instance, if the price is just a bit less than 0.404944e6, then technically, to reach this price point, the function will have to go through 1.006758e6, 0.885627e6 and 0.59332e6. This makes the other price checks extremely unnecessary.

```solidity
function getRatiosFromPriceLiquidity(uint256 price) external pure returns (PriceData memory) {
    if (price < 1.006758e6) {
        if (price < 0.885627e6) {
            if (price < 0.59332e6) {
                if (price < 0.404944e6) {
//...
```

#### Recommended Mitigation Steps

Recommend refactoring the logic, a potential way of doing this is using if method and rearranging each of the various price points in ascending order. If one isn't this, it can be skipped.

For example.
```solidity
if (price < 0.001083e6) {revert("LUT: Invalid price");}
if (price < 0.27702e6)  {return PriceData(0.27702e6, 0, 9.646293093274934449e18, 0.001083e6, 0, 2000e18, 1e18);}
if (price < 0.30624e6)  {return PriceData(0.30624e6, 0, 8.612761690424049377e18, 0.27702e6, 0, 9.646293093274934449e18, 1e18);}
if (price < 0.337394e6) {return PriceData(0.337394e6, 0, 7.689965795021471706e18, 0.30624e6, 0, 8.612761690424049377e18, 1e18);}
if (price < 0.370355e6) {return PriceData(0.370355e6, 0, 6.866040888412029197e18, 0.337394e6, 0, 7.689965795021471706e18, 1e18);}
if (price < 0.404944e6) {return PriceData(0.404944e6, 0, 6.130393650367882863e18, 0.370355e6, 0, 6.866040888412029197e18, 1e18);}
```

***
### 3. Add checks for same paramter updates

Links to affected code *

https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L63-L91

#### Impact

`_authorizeUpgrade` should check that `newImplementation` is not the same as previous iplementation.

```solidity
    function _authorizeUpgrade(address newImplementation) internal view override onlyOwner {
        // verify the function is called through a delegatecall.
        require(address(this) != ___self, "Function must be called through delegatecall");

        // verify the function is called through an active proxy bored by an aquifer.
        address aquifer = aquifer();
        address activeProxy = IAquifer(aquifer).wellImplementation(_getImplementation());
        require(activeProxy == ___self, "Function must be called through active proxy bored by an aquifer");

        // verify the new implmentation is a well bored by an aquifier.
        require(
            IAquifer(aquifer).wellImplementation(newImplementation) != address(0),
            "New implementation must be a well implmentation"
        );

        // verify the new well uses the same tokens in the same order.
        IERC20[] memory _tokens = tokens();
        IERC20[] memory newTokens = WellUpgradeable(newImplementation).tokens();
        require(_tokens.length == newTokens.length, "New well must use the same number of tokens");
        for (uint256 i; i < _tokens.length; ++i) {
            require(_tokens[i] == newTokens[i], "New well must use the same tokens in the same order");
        }

        // verify the new implmentation is a valid ERC-1967 implmentation.
        require(
            UUPSUpgradeable(newImplementation).proxiableUUID() == _IMPLEMENTATION_SLOT,
            "New implementation must be a valid ERC-1967 implmentation"
        );
    }
```

***


### 4. Change modifier name

Links to affected code *

https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L22

#### Impact

`notDelegatedOrIsMinimalProxy` modifier only ensures that call is through a minimal proxy. It doesn't ensure that the call is not delegated. Since this is the intended design, recommend renaming to something more conventional. 

```solidity
    modifier notDelegatedOrIsMinimalProxy() {
        if (address(this) != ___self) {
            address aquifer = aquifer();
            address wellImplmentation = IAquifer(aquifer).wellImplementation(address(this));
            require(wellImplmentation == ___self, "Function must be called by a Well bored by an aquifer");
        }
        _;
    }
```


***

### 5. Converging solution formula described in function helper is not same as implementation.

Links to affected code *

#### Impact

In helper string [here](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L73), converging solution is described as following.
```
D[j+1] = (4 * A * sum(b_i) - (D[j] ** 3) / (4 * prod(b_i))) / (4 * A - 1)       [1]
``` 
Invariant condition is as following:
$$An^n\sum x_i + D = DAn^n+D^{(n+1)}/(n^n\prod x_i)$$
```
A * sum(x_i) * n**n + D = A * D * n**n + D**(n+1) / (n**n * prod(x_i))
```

@Brean shared an article how someone derived D[j+1] from above invariant condition [here](https://atulagarwal.dev/posts/curveamm/stableswap/).
$$D_{(n+1)}=D_n-f(D_n)/f'(D_n)=(AnnS+nD_p)D_n/((Ann-1)D_n+(n+1)D_p)$$
where $$S=\sum x_i$$,  $$Ann=An^n$$,       $$D_p=D^{(n+1)}/(n^n \prod x_i)$$

As $$N=2$$, above equation can be simplified as following:
```
D[j+1]=(4*A*sum(b_i)+2*D_p)*D[j] / ( (4 * A - 1)*D[j] + (2+1)*D_p )       [2]
```  
Implementation [here](https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/functions/Stable2.sol#L94) is not same as [1], but same to [2]. 

#### Recommended Mitigation Steps

```diff
     * @notice Calculate the amount of lp tokens given reserves.
     * D invariant calculation in non-overflowing integer operations iteratively
     * A * sum(x_i) * n**n + D = A * D * n**n + D**(n+1) / (n**n * prod(x_i))
     *
     * Converging solution:
-    * D[j+1] = (4 * A * sum(b_i) - (D[j] ** 3) / (4 * prod(b_i))) / (4 * A - 1)
+    * D[j+1]=(4*A*sum(b_i)+2*D_p)*D[j] / ( (4 * A - 1)*D[j] + (2+1)*D_p )
```

***

### 6. `Implementation` is spelt `Implmentation`

Links to affected code *

https://github.com/code-423n4/2024-08-basin/blob/7e08ff591df0a2ade7d5618113dda2621cd899bc/src/WellUpgradeable.sol#L22-L30

#### Impact

```solidity
    modifier notDelegatedOrIsMinimalProxy() {
        if (address(this) != ___self) {
            address aquifer = aquifer();
            address wellImplmentation = IAquifer(aquifer).wellImplementation(address(this));
            require(wellImplmentation == ___self, "Function must be called by a Well bored by an aquifer");
        }
        _;
    }
```

Other instances are in comments and error messages.
```solidity
    // This pattern breaks the UUPS upgrade pattern, as the `__self` variable is set to the initial well implmentation.  ///<<== here
    //...
    // verification is done by verifying the ERC1967 implmentation (the well address) maps to the aquifers well -> implmentation mapping. ///<<== here
    //...
     * a proxy contract with an ERC1167 minimal proxy from an aquifier, pointing to a well implmentation.  ///<<== here
    //...
        // verify the new implmentation is a well bored by an aquifier.   ///<<== here
        require(
            IAquifer(aquifer).wellImplementation(newImplementation) != address(0),
            "New implementation must be a well implmentation"  ///<<== here
        );
    //...
        // verify the new implmentation is a valid ERC-1967 implmentation. ///<<== here
        require(
            UUPSUpgradeable(newImplementation).proxiableUUID() == _IMPLEMENTATION_SLOT,
            "New implementation must be a valid ERC-1967 implmentation" ///<<== here
        );
    }
```

#### Recommended Mitigation Steps

Update the spelling.
***

