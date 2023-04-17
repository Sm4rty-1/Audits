# Findings:
| Serial No. | Findings | Severity | Link to Original Report |
|-|-|-|-|
01. | Attacker can manipulate the myLink token's pricePerShare to take an unfair share | High | [Link](https://github.com/sherlock-audit/2022-10-mycelium-judging/blob/7babd1863dd7a557c128161957073f54f72e1370/001-H/117.md)


## A malicious early user/attacker can manipulate the myLink token's pricePerShare to take an unfair share of future users' deposits

### Summary
A well known attack vector for almost all shares based liquidity pool contracts, where an early user can manipulate the price per share and profit from late users' deposits because of the precision loss caused by the rather large value of price per share.

### Vulnerability Detail
A malicious early user can deposit() with 1 wei of asset token as the first depositor of the Token, and get 1 wei of shares.

Then the attacker can send 10000e18 - 1 of asset tokens and inflate the price per share from 1.0000 to an extreme value of 1.0000e22 ( from (1 + 10000e18 - 1) / 1) .

As a result, the future user who deposits 19999e18 will only receive 1 wei (from 19999e18 * 1 / 10000e18) of shares token.

They will immediately lose 9999e18 or half of their deposits if they redeem() right after the deposit().


### Impact
The attacker can profit from future users' deposits. While the late users will lose part of their funds to the attacker.

### Code Snippet
https://github.com/sherlock-audit/2022-10-mycelium/blob/main/mylink-contracts/src/Vault.sol#L131
```solidity
    function deposit(uint256 _amount) external {
        require(_amount > 0, "Amount must be greater than 0");
        require(_amount <= availableForDeposit(), "Amount exceeds available capacity");

        uint256 newShares = convertToShares(_amount);
        _mintShares(msg.sender, newShares);

        IERC20(LINK).transferFrom(msg.sender, address(this), _amount);
        _distributeToPlugins();

        emit Deposit(msg.sender, _amount);
    }
```
https://github.com/sherlock-audit/2022-10-mycelium/blob/main/mylink-contracts/src/Vault.sol#L614
```solidity
    function convertToShares(uint256 _tokens) public view returns (uint256) {
        uint256 tokenSupply = totalSupply(); // saves one SLOAD
        if (tokenSupply == 0) {
            return _tokens * STARTING_SHARES_PER_LINK;
        }
        return _tokens.mulDivDown(totalShares, tokenSupply);
    }
```

### Tool used
Manual Review

### Recommendation
Consider requiring a minimal amount of share tokens to be minted for the first minter, and send a port of the initial mints as a reserve to the DAO so that the pricePerShare can be more resistant to manipulation.

A similar issue found in recent Sherlock Contests:
https://github.com/sherlock-audit/2022-08-sentiment-judging/blob/main/004-H/1-report.md