### [H-#] erroneous `Thunder::updateExchageRate` in the `update` causes protocol to think it has collected more fees than it really has, which blocks redeemption andincorrectly sets the exchange rate

**Description:** In the ThunderLoan system, the `exchangeRate` is responsible for calculating the exchage rate between assetTokens and underlying tokens. In a way , it's responsible for keeping track of how many fees to give to liquidity providers.

However, the `deposit` function, updates the rate without collecting any fees! 

```javascript
  function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
        uint256 calculatedFee = getCalculatedFee(token, amount);
        assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }

```
**Impact:** There is more bugs to this function

1. The redeem function is blocked because the protocol thinks the owned tokens is more that it has.
2. Rewards are incorrectly calculated which means that uses get way less or more than what they deserve.

**Proof of concept:** 
1. LP deposits
2. User takes out a flash loan
3. It is now impossible for Lp to redeem.

<details>

<summary>Proof of code</summary>

Place the following into ThunderloanTest.t.sol

```javascript
function testRedeemAfterFlashLoan() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), AMOUNT);
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA, amountToRedeem);

    }

```

</details>

**Recommended Mitigation:** Remove the incorrect updated exchage tate lines from `deposit`

```diff
  function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
-        uint256 calculatedFee = getCalculatedFee(token, amount);
-        assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

### [M - 2] Using Tswap as price oracle leads to price and oracle manupulation attack.

**Description:**  The Tswap protocol is a constant product formula based AMM (automated market maker). The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token buying and selling a large amount of the token in the same transaction, essentially ignoring protocol fees.

**Impact:** Liquidity providers get drastically reduced fees for providing liquidity.

**Proof of Concept:**

All the code below happens in one transaction

1. User takes a flash loan from `ThunderLoan` for 1000 `tokenA`. they are charged the original fee `fee1`. During the flash loan, they do the following

        1. user sells 1000 `tokenA`, tanking the price,
        2. instead of repaying right away the user takes out another flash loan for another 1000 `tokenA`.
           1. Due to the fact that the way `ThunderLoan` calculates price based on `TswapPool` this second flash Loan is substantially cheaper


```javascript
function getPriceInWeth(address token) public view returns (uint256) {
        address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
        return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
    }

        3. then the user repays the first flashloan and repays  the second flash loan

I have created a POC in the ThunderloanTest.t.sol testOraclemanipulation

**Recommended MItigation:** Consider using a different price oracle mechanism like a chainlink price feed with a Uniswap TWAP fallback oracle.



```