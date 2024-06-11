# REPORT - Vault Guardians

### Issue Summary

| Category          | No. of Issues |
| ----------------- | ------------- |
| High              | 5             |
| Medium            | 0             |
| Low               | 6             |
| Informational/Gas | 2             |

## HIGH

### [H-1] Incorrect Investment Calculation in `UniswapAdapter::_uniswapInvest` (Logic Flaw -> Overinvestment)

**Description:**

The `UniswapAdapter::_uniswapInvest` function is designed to invest half of the provided amount in WETH and the other half in another token. However, due to a logical error, it incorrectly adds `amounts[0]` to the amount being invested, resulting in an overinvestment in the Uniswap liquidity pool.

**Impact:**

This bug causes the contract to invest more funds than intended in the Uniswap liquidity pool. This can lead to the misallocation of assets, reducing the available liquidity in the vault and potentially affecting the overall performance of the investment strategy. Users may also receive fewer shares than they should, leading to potential financial discrepancies and trust issues.

**Proof of Concept:**

The problematic code is located in the `_uniswapInvest` function:

```javascript
function _uniswapInvest(IERC20 token, uint256 amount) internal {
    IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;
        // We will do half in WETH and half in the token
        uint256 amountOfTokenToSwap = amount / 2;
        s_pathArray = [address(token), address(counterPartyToken)];

        bool succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap);
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }
        uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({
            amountIn: amountOfTokenToSwap,
            amountOutMin: 0,
            path: s_pathArray,
            to: address(this),
            deadline: block.timestamp
        });

        succ = counterPartyToken.approve(address(i_uniswapRouter), amounts[1]);
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }

        succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap + amounts[0]); // <- Problematic line
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }

        (uint256 tokenAmount, uint256 counterPartyTokenAmount, uint256 liquidity) = i_uniswapRouter.addLiquidity({
            tokenA: address(token),
            tokenB: address(counterPartyToken),
            amountADesired: amountOfTokenToSwap + amounts[0], // <- Problematic line
            amountBDesired: amounts[1],
            amountAMin: amountOfTokenToSwap,
            amountBMin: 0,
            to: address(this),
            deadline: block.timestamp
        });
        emit UniswapInvested(tokenAmount, counterPartyTokenAmount, liquidity);
}
```

The line `succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap + amounts[0]);` mistakenly adds amounts[0] to amountOfTokenToSwap, leading to an incorrect approval and overinvestment when calling i_uniswapRouter.addLiquidity.

Place the following code into VaultSharesTest.t.sol:

```javascript
function testUniswapInvestWrong() public hasGuardian {
    weth.mint(1000, user);
    vm.startPrank(user);
    weth.approve(address(wethVaultShares), 1000);
    uint256 startingBalance = weth.balanceOf(address(wethVaultShares));
    wethVaultShares.deposit(1000, guardian);
    console.log(weth.balanceOf(address(wethVaultShares)));
    // @note should be higher for 500 according to allocationData,
    // but it is higher for only 500-250/2 due
    // to the _uniswapInvest and its wrong logic!
    assertEq(weth.balanceOf(address(wethVaultShares)), startingBalance + 500 - 250 / 2);
}
```

The test case shows that the WETH balance of the vault is lower than expected due to the incorrect logic in the \_uniswapInvest function.

**Recommended Mitigation:**
To fix this issue, you could do the following:

```diff
function _uniswapInvest(IERC20 token, uint256 amount) internal {
    IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;
        // We will do half in WETH and half in the token
        uint256 amountOfTokenToSwap = amount / 2;
        s_pathArray = [address(token), address(counterPartyToken)];

        bool succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap);
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }
        uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({
            amountIn: amountOfTokenToSwap,
            amountOutMin: 0,
            path: s_pathArray,
            to: address(this),
            deadline: block.timestamp
        });

        succ = counterPartyToken.approve(address(i_uniswapRouter), amounts[1]);
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }

-       succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap + amounts[0]); // <- Problematic line
+       succ = token.approve(address(i_uniswapRouter), amountOfTokenToSwap);
        if (!succ) {
            revert UniswapAdapter__TransferFailed();
        }

        (uint256 tokenAmount, uint256 counterPartyTokenAmount, uint256 liquidity) = i_uniswapRouter.addLiquidity({
            tokenA: address(token),
            tokenB: address(counterPartyToken),
-           amountADesired: amountOfTokenToSwap + amounts[0], // <- Problematic line
+           amountADesired: amountOfTokenToSwap,
            amountBDesired: amounts[1],
            amountAMin: amountOfTokenToSwap,
            amountBMin: 0,
            to: address(this),
            deadline: block.timestamp
        });
        emit UniswapInvested(tokenAmount, counterPartyTokenAmount, liquidity);
}
```

This ensures that only the intended amount is approved for investment, preventing overinvestment in the Uniswap liquidity pool.

### [H-2] In the `VaultShares::deposit()` function, we are issuing more shares to users than we should, which will lead to an imbalance between the invested amount of capital and the shares as proof of the invested amount.

**Description:**

In the `VaultShares::deposit()` function, when a user deposits a certain amount of funds into the vault, we mint a certain number of shares as proof that the user has invested those funds. Later, based on these shares, the user can withdraw their funds back. During this process, a portion of the shares is also minted for the guardian and VaultGuardian (based on the user's invested funds).

```javascript
    uint256 shares = previewDeposit(assets);
    _deposit(_msgSender(), receiver, assets, shares);
    _mint(i_guardian, shares / i_guardianAndDaoCut);
    _mint(i_vaultGuardians, shares / i_guardianAndDaoCut);
```

As we can see from the provided information, the entire value of shares calculated based on the user's invested funds is sent to the user. Then, a portion is sent to the guardian and VaultGuardian. The problem is that the portion sent to them is not previously deducted from the total shares sent to the user.

**Impact:**

This will result in issuing more shares than there are funds in the vault. In this way, we create an imbalance between the amount of funds in the vault and the shares as proof of that amount.

**Recommended Mitigation:**

One of the possible ways to correct this error is given below:

```diff
    function deposit(uint256 assets, address receiver)
        public
        override(ERC4626, IERC4626)
        isActive
        nonReentrant
        returns (uint256)
    {
        if (assets > maxDeposit(receiver)) {
            revert VaultShares__DepositMoreThanMax(assets, maxDeposit(receiver));
        }

        //q treba li da provjerava vrijednost za shares?
        uint256 shares = previewDeposit(assets);
-       _deposit(_msgSender(), receiver, assets, shares);
+       _deposit(_msgSender(), receiver, assets, shares-shares / i_guardianAndDaoCut-shares / i_guardianAndDaoCut);
        _mint(i_guardian, shares / i_guardianAndDaoCut);
        _mint(i_vaultGuardians, shares / i_guardianAndDaoCut);

        _investFunds(assets);
        return shares;
    }
```

### [H-3] Lack of UniswapV2 slippage protection in `UniswapAdapter::_uniswapInvest` enables frontrunners to steal profits

**Description:**

In `UniswapAdapter::_uniswapInvest` the protocol swaps half of an ERC20 token so that they can invest in both sides of a Uniswap pool. It calls the `swapExactTokensForTokens` function of the `UnisapV2Router01` contract , which has two input parameters to note:

```javascript
    function swapExactTokensForTokens(
        uint256 amountIn,
@>      uint256 amountOutMin,
        address[] calldata path,
        address to,
@>      uint256 deadline
    )
```

The parameter `amountOutMin` represents how much of the minimum number of tokens it expects to return.
The `deadline` parameter represents when the transaction should expire.

As seen below, the `UniswapAdapter::_uniswapInvest` function sets those parameters to `0` and `block.timestamp`:

```javascript
    uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens(
        amountOfTokenToSwap,
@>      0,
        s_pathArray,
        address(this),
@>      block.timestamp
    );
```

**Impact:**

This results in either of the following happening:

- Anyone (e.g., a frontrunning bot) sees this transaction in the mempool, pulls a flashloan and swaps on Uniswap to tank the price before the swap happens, resulting in the protocol executing the swap at an unfavorable rate.
- Due to the lack of a deadline, the node who gets this transaction could hold the transaction until they are able to profit from the guaranteed swap.

**Proof of Concept:**

1. User calls `VaultShares::deposit` with a vault that has a Uniswap allocation.
   1. This calls `_uniswapInvest` for a user to invest into Uniswap, and calls the router's `swapExactTokensForTokens` function.
2. In the mempool, a malicious user could:
   1. Hold onto this transaction which makes the Uniswap swap
   2. Take a flashloan out
   3. Make a major swap on Uniswap, greatly changing the price of the assets
   4. Execute the transaction that was being held, giving the protocol as little funds back as possible due to the `amountOutMin` value set to 0.

This could potentially allow malicious MEV users and frontrunners to drain balances.

**Recommended Mitigation:**

_For the deadline issue, we recommend the following:_

DeFi is a large landscape. For protocols that have sensitive investing parameters, add a custom parameter to the `deposit` function so the Vault Guardians protocol can account for the customizations of DeFi projects that it integrates with.

In the `deposit` function, consider allowing for custom data.

```diff
- function deposit(uint256 assets, address receiver) public override(ERC4626, IERC4626) isActive returns (uint256) {
+ function deposit(uint256 assets, address receiver, bytes customData) public override(ERC4626, IERC4626) isActive returns (uint256) {
```

This way, you could add a `deadline` to the Uniswap swap, and also allow for more DeFi custom integrations.

_For the `amountOutMin` issue, we recommend one of the following:_

1. Do a price check on something like a [Chainlink price feed](https://docs.chain.link/data-feeds) before making the swap, reverting if the rate is too unfavorable.
2. Only deposit 1 side of a Uniswap pool for liquidity. Don't make the swap at all. If a pool doesn't exist or has too low liquidity for a pair of ERC20s, don't allow investment in that pool.

Note that these recommendation require significant changes to the codebase.

### [H-4] Guardians can infinitely mint `VaultGuardianToken`s and take over DAO, stealing DAO fees and maliciously setting parameters

**Description:**

Becoming a guardian comes with the perk of getting minted Vault Guardian Tokens (vgTokens). Whenever a guardian successfully calls `VaultGuardiansBase::becomeGuardian` or `VaultGuardiansBase::becomeTokenGuardian`, `_becomeTokenGuardian` is executed, which mints the caller `i_vgToken`.

```javascript
    function _becomeTokenGuardian(IERC20 token, VaultShares tokenVault) private returns (address) {
        s_guardians[msg.sender][token] = IVaultShares(address(tokenVault));
@>      i_vgToken.mint(msg.sender, s_guardianStakePrice);
        emit GuardianAdded(msg.sender, token);
        token.safeTransferFrom(msg.sender, address(this), s_guardianStakePrice);
        token.approve(address(tokenVault), s_guardianStakePrice);
        tokenVault.deposit(s_guardianStakePrice, msg.sender);
        return address(tokenVault);
    }
```

Guardians are also free to quit their role at any time, calling the `VaultGuardianBase::quitGuardian` function. The combination of minting vgTokens, and freely being able to quit, results in users being able to farm vgTokens at any time.

**Impact:**

Assuming the token has no monetary value, the malicious guardian could accumulate tokens until they can overtake the DAO. Then, they could execute any of these functions of the `VaultGuardians` contract:

```
  "sweepErc20s(address)": "942d0ff9",
  "transferOwnership(address)": "f2fde38b",
  "updateGuardianAndDaoCut(uint256)": "9e8f72a4",
  "updateGuardianStakePrice(uint256)": "d16fe105",
```

**Proof of Concept:**

1. User becomes WETH guardian and is minted vgTokens.
2. User quits, is given back original WETH allocation.
3. User becomes WETH guardian with the same initial allocation.
4. Repeat to keep minting vgTokens indefinitely.

<details>
<summary>Code</summary>

Place the following code into `VaultGuardiansBaseTest.t.sol`

```javascript
    function testDaoTakeover() public hasGuardian hasTokenGuardian {
        address maliciousGuardian = makeAddr("maliciousGuardian");
        uint256 startingVoterUsdcBalance = usdc.balanceOf(maliciousGuardian);
        uint256 startingVoterWethBalance = weth.balanceOf(maliciousGuardian);
        assertEq(startingVoterUsdcBalance, 0);
        assertEq(startingVoterWethBalance, 0);

        VaultGuardianGovernor governor = VaultGuardianGovernor(payable(vaultGuardians.owner()));
        VaultGuardianToken vgToken = VaultGuardianToken(address(governor.token()));

        // Flash loan the tokens, or just buy a bunch for 1 block
        weth.mint(mintAmount, maliciousGuardian); // The same amount as the other guardians
        uint256 startingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 startingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious vgToken Balance:\t", startingMaliciousVGTokenBalance);
        console.log("Regular vgToken Balance:\t", startingRegularVGTokenBalance);

        // Malicious Guardian farms tokens
        vm.startPrank(maliciousGuardian);
        weth.approve(address(vaultGuardians), type(uint256).max);
        for (uint256 i; i < 10; i++) {
            address maliciousWethSharesVault = vaultGuardians.becomeGuardian(allocationData);
            IERC20(maliciousWethSharesVault).approve(
                address(vaultGuardians),
                IERC20(maliciousWethSharesVault).balanceOf(maliciousGuardian)
            );
            vaultGuardians.quitGuardian();
        }
        vm.stopPrank();

        uint256 endingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 endingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious vgToken Balance:\t", endingMaliciousVGTokenBalance);
        console.log("Regular vgToken Balance:\t", endingRegularVGTokenBalance);
    }
```

</details>

**Recommended Mitigation:** There are a few options to fix this issue:

1. Mint vgTokens on a vesting schedule after a user becomes a guardian.
2. Burn vgTokens when a guardian quits.
3. Simply don't allocate vgTokens to guardians. Instead, mint the total supply on contract deployment.

### [H-5] In the `UniswapAdapter::_uniswapInvest` function, `amounts[1]` is not always WETH as you expect

**Description:**

In the `_uniswapInvest` function, `amounts[1]` represents the amount of the counterparty token obtained during the swap. This amount is not always WETH; it depends on the input token (`token`). If the input `token` is `i_weth`, `amounts[1]` will be the amount of `i_tokenOne`. Conversely, if the input `token` is `i_tokenOne`, `amounts[1]` will be WETH.

**Impact:**

Assuming `amounts[1]` is always WETH can lead to incorrect calculations and potential errors during token approval and liquidity addition. This could result in failed transactions or loss of funds.

**Proof of Concept:**

1. Consider the scenario where `token` is equal to `i_weth`:

   ```javascript
   IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;
   ```

   In this case, `counterPartyToken` will be `i_tokenOne`.

2. During the token swap:

   ```javascript
   uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({
       amountIn: amountOfTokenToSwap,
       amountOutMin: 0,
       path: s_pathArray,
       to: address(this),
       deadline: block.timestamp
   });
   ```

   `amounts[1]` will be the amount of `i_tokenOne` received in exchange for `i_weth`.

3. If `token` is not `i_weth` (e.g., `i_tokenOne`):

   ```javascript
   IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;
   ```

   `counterPartyToken` will be `i_weth`.

4. During the token swap:
   ```javascript
   uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({
       amountIn: amountOfTokenToSwap,
       amountOutMin: 0,
       path: s_pathArray,
       to: address(this),
       deadline: block.timestamp
   });
   ```
   `amounts[1]` will be the amount of WETH received in exchange for `i_tokenOne`.

**Recommended Mitigation:**

1. **Check the type of received token:** After the token swap, check the type of the received token (`amounts[1]`) to determine if it is WETH or another token.

   ```javascript
   bool isWETH = counterPartyToken == i_weth;
   ```

Implementing this measure will ensure accuracy during token approval and liquidity addition, avoiding errors and potential losses.

## LOW

### [L-1] Missing Event in `AaveAdapter::_aaveDivest` (Lack of Event -> Reduced Transparency)

**Description:**

In the `AaveAdapter` contract, the `_aaveDivest` function performs significant actions such as withdrawing assets from the Aave pool. However, it does not emit any events to log these actions, which reduces transparency and makes it difficult to track the function's operations.

**Impact:**

The absence of events in the `_aaveDivest` function leads to reduced transparency and hampers the ability to audit and monitor the contract's activities. This can pose challenges in debugging and understanding the flow of funds, especially for significant operations like asset withdrawals.

**Code Example:**

The `_aaveDivest` function is defined as follows in the `AaveAdapter` contract:

```javascript
function _aaveDivest(IERC20 token, uint256 amount) internal returns (uint256 amountOfAssetReturned) {
    i_aavePool.withdraw({
        asset: address(token),
        amount: amount,
        to: address(this)
    });
    //q missing events
}
```

**Recommended Mitigation**

Consider adding an event to log the details of the asset withdrawal. For example:

```javascript
event AaveDivested(address indexed token, uint256 amount);
function _aaveDivest(IERC20 token, uint256 amount) internal returns (uint256 amountOfAssetReturned) {
    uint256 amountOfAssetReturned = i_aavePool.withdraw({
        asset: address(token),
        amount: amount,
        to: address(this)
    });
    emit AaveDivested(address(token), amount);
}
```

Adding such an event enhances transparency and allows for better monitoring and auditing of the contract's operations.

### [L-2] Missing Event in `AaveAdapter::_aaveInvest` (Lack of Event -> Reduced Transparency)

**Description:**

In the `AaveAdapter` contract, the `_aaveInvest` function performs significant actions such as supplying assets to the Aave pool. However, it does not emit any events to log these actions, which reduces transparency and makes it difficult to track the function's operations.

**Impact:**

The absence of events in the `_aaveInvest` function leads to reduced transparency and hampers the ability to audit and monitor the contract's activities. This can pose challenges in debugging and understanding the flow of funds, especially for significant operations like asset investments.

**Code Example:**

The `_aaveInvest` function is defined as follows in the `AaveAdapter` contract:

```javascript
function _aaveInvest(IERC20 asset, uint256 amount) internal {
    bool succ = asset.approve(address(i_aavePool), amount);
    if (!succ) {
        revert AaveAdapter__TransferFailed();
    }
    i_aavePool.supply({
        asset: address(asset),
        amount: amount,
        onBehalfOf: address(this),
        referralCode: 0
    });
    //q missing events
}
```

**Recommended Mitigation**

Consider adding an event to log the details of the asset supply. For example:

```javascript
event AaveInvested(address indexed asset, uint256 amount);

function _aaveInvest(IERC20 asset, uint256 amount) internal {
    bool succ = asset.approve(address(i_aavePool), amount);
    if (!succ) {
        revert AaveAdapter__TransferFailed();
    }
    i_aavePool.supply({
        asset: address(asset),
        amount: amount,
        onBehalfOf: address(this),
        referralCode: 0
    });
    emit AaveInvested(address(asset), amount);
}

```

Adding such an event enhances transparency and allows for better monitoring and auditing of the contract's operations.

### [L-3] In the `VaultGuardian::updateGuardianStakePrice()` function, the values of the old and new prices for becoming a guardian (StakePrice) are emitted incorrectly.

**Description:**

In the function that is mentioned above, event is emited with the same value of old and new StakePrice. This is happening due to the fact that the value is updated before emiting an event.

**Impact:**

As a result of this, we will have confusing information on the blockchain about the old and new StakePrice.

**Recommended Mitigation:**

The given problem can be overcome in the following way:

```diff
    function updateGuardianStakePrice(uint256 newStakePrice) external onlyOwner {
+       emit VaultGuardians__UpdatedStakePrice(s_guardianStakePrice, newStakePrice);
        s_guardianStakePrice = newStakePrice;
-       emit VaultGuardians__UpdatedStakePrice(s_guardianStakePrice, newStakePrice);
    }
```

### [L-4] In the `VaultGuardians::updateGuardianAndDaoCut()` function, the wrong event is emitted.

**Description:**

In the `VaultGuardians::updateGuardianAndDaoCut()` function, which is used to update the value based on which a portion of the user's deposited funds is allocated to the guardian and VaultGuardian, the wrong event is emitted. The incorrectly emitted event is VaultGuardians\_\_UpdatedStakePrice(s_guardianAndDaoCut, newCut), which is actually intended for emitting when updating the stakePrice.

**Impact:**

As a result of this, we will have incorrect information on the blockchain about the earnings of the guardian and VaultGuardian (GuardianAndDaoCut), as well as the value required to become a guardian (StakePrice).

**Recommended Mitigation:**

Instead of the given event, it is necessary to emit the VaultGuardians\_\_UpdatedFee(uint256 oldFee, uint256 newFee) event:

```diff
    function updateGuardianAndDaoCut(uint256 newCut) external onlyOwner {
+       emit VaultGuardians\_\_UpdatedFee(s_guardianAndDaoCut,newCut);
        s_guardianAndDaoCut = newCut;
-       emit VaultGuardians__UpdatedStakePrice(s_guardianAndDaoCut, newCut);
    }
```

### [L-5] Incorrect vault name and symbol

In the VaultGuardianBase::becomeTokenGuardian function, when forming a new vault, if `token == i_tokenOne`, the name and symbol are set incorrectly. This will provide users with incorrect information about the token, potentially causing confusion.

The following modification to the given function code is necessary:

```diff
else if (address(token) == address(i_tokenTwo)) {
    tokenVault =
    new VaultShares(IVaultShares.ConstructorData({
        asset: token,
+       vaultName: TOKEN_TWO_VAULT_NAME,
-       vaultName: TOKEN_ONE_VAULT_NAME,
+       vaultSymbol: TOKEN_TWO_VAULT_SYMBOL,
-       vaultSymbol: TOKEN_ONE_VAULT_SYMBOL,
        guardian: msg.sender,
        allocationData: allocationData,
        aavePool: i_aavePool,
        uniswapRouter: i_uniswapV2Router,
        guardianAndDaoCut: s_guardianAndDaoCut,
        vaultGuardian: address(this),
        weth: address(i_weth),
        usdc: address(i_tokenOne)
    }));
```

### [L-6] `AaveAdapter::_aaveDivest` function does not return expected value

**Description:**

The `_aaveDivest` function is intended to withdraw a specified amount of a token from the Aave pool and return the amount of the asset withdrawn. However, the function does not actually return any value, despite being defined to return `uint256 amountOfAssetReturned`.

**Impact:**

The missing return value can lead to incorrect assumptions and potential errors in the calling functions that expect `amountOfAssetReturned` to contain the withdrawn amount. This could result in logic errors, miscalculations, and unintended behaviors in the broader system relying on this function.

**Recommended Mitigation:**

To fix this issue, capture the return value of the `withdraw` function and explicitly return it from `_aaveDivest`:

```javascript
function _aaveDivest(IERC20 token, uint256 amount) internal returns (uint256 amountOfAssetReturned) {
    amountOfAssetReturned = i_aavePool.withdraw({
        asset: address(token),
        amount: amount,
        to: address(this)
    });
    return amountOfAssetReturned;
}
```

## Informational/Gas

### [I/G-1] Unused Constant Variable `GUARDIAN_FEE` (Unused Variable -> Increased Gas Costs)

**Description:**

In the `VaultGuardiansBase` contract, there is a constant variable `GUARDIAN_FEE` defined but not used anywhere in the contract. This unnecessary constant increases the gas cost for deployment.

**Impact:**

Since `GUARDIAN_FEE` is a constant, it contributes to the overall gas cost when deploying the contract. Even though it is not used, its presence leads to higher deployment costs, which could be avoided if the variable were removed.

**Proof of Concept:**

The unused constant is defined as follows in the `VaultGuardiansBase` contract:

```javascript
uint256 private constant GUARDIAN_FEE = 0.1 ether; // unused variable! GAS PROBLEM
```

However, there are no references to GUARDIAN_FEE in the rest of the protocol, indicating that it serves no functional purpose.
By including this constant without utilizing it, the contract incurs unnecessary gas costs during deployment.

### [I/G-2] Unused Events `InvestedInGuardian` and `DinvestedFromGuardian`

**Description:**

In the `VaultGuardiansBase` contract, there are two events, `InvestedInGuardian` and `DinvestedFromGuardian`, that are defined but not used anywhere in the contract.

**Proof of Concept:**

The unused events are defined as follows in the `VaultGuardiansBase` contract:

```javascript
//q unused event
event InvestedInGuardian(address guardianAddress, IERC20 token, uint256 amount);
//q unused event
event DinvestedFromGuardian(address guardianAddress, IERC20 token, uint256 amount);
```

However, there are no references to InvestedInGuardian and DinvestedFromGuardian in the rest of the protocol, indicating that they serve no functional purpose.
