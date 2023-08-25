## Sekizo - [contract address](https://etherscan.io/address/0xf2680ac4b924642163bd700a8ce88e8fe29eba94#code)
### 1. Introduction
This malicious contract incorporates a total of five tricks, comprising four trapdoors cleverly disguised within the conditional checking category, along with one trapdoor skillfully camouflaged within the fee manipulation category. 

### 2. Analysis
- **_Disabling Trading (Conditional checking):_**
  
  _The variable "isTradingEnabled" can be modified by the contract creator to the value false. This action effectively halts token transactions from investors._
  
- **_Blacklist checking (Conditional checking):_**
  
  _The variable_ "_isBlocked" is a map designed to store addresses. Its population is restricted to the creator of the contract, granting them the exclusive ability to utilize it as a mechanism for preventing investors from selling their tokens._
  
- **_invalid limitation update from creator to maximum transaction amount (Conditional checking):_**
  
  _The variable "maxTxAmount" can be modified by the contract creator to the value of zero. This action effectively halts token transactions from investors._
  
- **_invalid limitation update from creator to maximum wallet amount (Conditional checking):_**
  
  _The variable "maxWalletAmount" can be modified by the contract creator to the value of zero. This action effectively halts token transactions from investors._

- **_manoeuvring fees to 25% on a speacial case (Fee manipulation):_**
  
  _The "\_launchTimestamp" variable and the activeTrading function can be utilized to impose a malicious fee of 25% on investors when they attempt to sell the token through Uniswap within the first three days of the contract deployment._

### 3. Explanation
- **_Disabling Trading (Conditional checking):_**

    ```solidity
    657    function _transfer(
    658    address from,
    659    address to,
    660    uint256 amount
    661    ) internal {
    662        require(from != address(0), "ERC20: transfer from the zero address");
    663        require(to != address(0), "ERC20: transfer to the zero address");
    664        require(amount > 0, "Transfer amount must be greater than zero");
    665        require(amount <= balanceOf(from), "The Revolution Token: Cannot transfer more than balance");
    666
    667        bool isBuyFromLp = automatedMarketMakerPairs[from];
    668        bool isSelltoLp = automatedMarketMakerPairs[to];
    669
    670        if(!_isAllowedToTradeWhenDisabled[from] && !_isAllowedToTradeWhenDisabled[to]) {
    671            require(isTradingEnabled, "The Revolution Token: Trading is currently disabled.");
    672            require(!_isBlocked[to], "The Revolution Token: Account is blocked");
    673            require(!_isBlocked[from], "The Revolution Token: Account is blocked");
    674            if (!_isExcludedFromMaxTransactionLimit[to] && !_isExcludedFromMaxTransactionLimit[from]) {
    675                require(amount <= maxTxAmount, "The Revolution Token: Transfer amount exceeds the maxTxAmount.");
    676            }
    677            if (!_isExcludedFromMaxWalletLimit[to]) {
    678                require((balanceOf(to) + amount) <= maxWalletAmount, "The Revolution Token: Expected wallet amount exceeds the maxWalletAmount.");
    679            }
    680        }
    681
    682        _adjustTaxes(isBuyFromLp, isSelltoLp);
    683        bool canSwap = balanceOf(address(this)) >= minimumTokensBeforeSwap;
    684
    685        if (
    686            isTradingEnabled &&
    687            canSwap &&
    688            !_swapping &&
    689            _totalFee > 0 &&
    690            automatedMarketMakerPairs[to]
    691        ) {
    692            _swapping = true;
    693            _swapAndLiquify();
    694            _swapping = false;
    695        }
    696
    697        bool takeFee = !_swapping && isTradingEnabled;
    698
    699        if(_isExcludedFromFee[from] || _isExcludedFromFee[to]){
    700            takeFee = false;
    701        }
    702        _tokenTransfer(from, to, amount, takeFee);
    703    }
    ```
  
  _In the Sekizo contract, both the transfer and transferFrom functions invoke the _transfer function. By default, line number 670 evaluates to true as the _isAllowedToTradeWhenDisabled map only includes the contract address and the owner’s address, and no other addresses are present._

     ```solidity
	494    function deactivateTrading() external onlyOwner {
	495    	isTradingEnabled = false;
	496    }
     ``` 

  _At line number 671, there is a require statement that checks the value of the isTradingEnabled variable. It ensures that the value is true in order for the following code to proceed. The owner of the contract holds the ability to modify this variable by employing the deactivateTrading function, effectively controlling as a sell restriction mechanism._
  
- **_Blacklist checking (Conditional checking):_**

    ```solidity
    502    function blockAccount(address account) external onlyOwner {
    503        require(!_isBlocked[account], "The Revolution Token: Account is already blocked");
    504        require((block.timestamp - _launchTimestamp) < _blockedTimeLimit, "The Revolution Token: Time to block accounts has expired");
    505        _isBlocked[account] = true;
    506        emit BlockedAccountChange(account, true);
    507    }
    ```
  
  _Lines 672 and 673 include two additional require statements that ensure both the from and to addresses are not present in the isBlocked map. The isBlocked map can be populated by the contract owner using the blockAccount function. It's important to note that this function is only available for execution within three days of deploying the contract. However, this limited time frame is sufficient for the owner to potentially carry out a malicious action._
   
  
- **_invalid limitation update from creator to maximum transaction amount (Conditional checking):_**

    ```solidity
    527    function excludeFromMaxTransactionLimit(address account, bool excluded) external onlyOwner {
    528        require(_isExcludedFromMaxTransactionLimit[account] != excluded, "The Revolution Token: Account is already the value of 'excluded'");
    529        _isExcludedFromMaxTransactionLimit[account] = excluded;
    530        emit ExcludeFromMaxTransferChange(account, excluded);
    531    }
    ```

    ```solidity
    568    function setMaxTransactionAmount(uint256 newValue) external onlyOwner {
    569        require(newValue != maxTxAmount, "The Revolution Token: Cannot update maxTxAmount to same value");
    570        emit MaxTransactionAmountChange(newValue, maxTxAmount);
    571        maxTxAmount = newValue;
    572    }    
    ```

   _Lines 674 to 676 encompass nested if conditions with a require statement that verifies if the transaction amount is less than the maxTxAmount. The contract owner holds the ability to add or remove values to the _isExcludedFromMaxTransactionLimit map using the excludeFromMaxTransactionLimit function. Furthermore, the owner can modify the value of the maxTxAmount variable using the setMaxTransactionAmount function. These two functionalities combined enable the contract owner to selectively permit specific addresses to sell their tokens while restricting other token holders from selling theirs._


- **_invalid limitation update from creator to maximum wallet amount (Conditional checking):_**

    ```solidity
    522    function excludeFromMaxWalletLimit(address account, bool excluded) external onlyOwner {
    523        require(_isExcludedFromMaxWalletLimit[account] != excluded, "The Revolution Token: Account is already the value of 'excluded'");
    524        _isExcludedFromMaxWalletLimit[account] = excluded;
    525        emit ExcludeFromMaxWalletChange(account, excluded);
    526    }
    ```

    _Similarly, lines 677 to 679 can be employed in a similar fashion as discussed above. The excludeFromMaxWalletLimit function enables the contract owner to modify the values in the _isExcludedFromMaxWalletLimit map, while the setMaxWalletAmount function allows for adjustment of the maxWalletAmount variable. This combination again serves as a sell restriction mechanism._


- **_manoeuvring fees to 25% on a speacial case (Fee manipulation):_**

    ```solidity
    771    function _adjustTaxes(bool isBuyFromLp, bool isSelltoLp) private {
    772        _liquidityFee = 0;
    773        _devFee = 0;
    774        _marketingFee = 0;
    775        _buyBackFee = 0;
    776        _holdersFee = 0;
    777
    778        if (isBuyFromLp) {
    779            if ((block.number - _launchBlockNumber) <= 5) {
    780                _liquidityFee = 100;
    781            } else {
    782                _liquidityFee = _base.liquidityFeeOnBuy;
    783                _devFee = _base.devFeeOnBuy;
    784                _marketingFee = _base.marketingFeeOnBuy;
    785                _buyBackFee = _base.buyBackFeeOnBuy;
    786                _holdersFee = _base.holdersFeeOnBuy;
    787            }
    788        }
    789        if (isSelltoLp) {
    790            _liquidityFee = _base.liquidityFeeOnSell;
    791            _devFee = _base.devFeeOnSell;
    792            _marketingFee = _base.marketingFeeOnSell;
    793            _buyBackFee = _base.buyBackFeeOnSell;
    794            _holdersFee = _base.holdersFeeOnSell;
    795
    796            if (block.timestamp - _launchTimestamp <= 259200) {
    797                _liquidityFee = 2;
    798                _devFee = 3;
    799                _marketingFee = 10;
    800                _buyBackFee = 8;
    801                _holdersFee = 2;
    802            }
    803        }
    804        _totalFee = _liquidityFee + _marketingFee + _devFee + _buyBackFee + _holdersFee;
    805        emit FeesApplied(_liquidityFee, _marketingFee, _devFee, _buyBackFee, _holdersFee, _totalFee);
    806    }
    ```

    _Lines 796 to 803 involve the assignment of fee variables. In a specific scenario where an investor sells to Uniswap within 3 days of deploying the contract, a significant malicious fee of 25% will be applied._
    
  
