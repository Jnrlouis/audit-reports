# Introduction

A time-boxed security review of the **Test Hacken Staking Contract** was done by **jnrlouis**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **jnrlouis**

Louis, or **jnrlouis**, is a dedicated and driven junior smart contract security researcher. He does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@Lo0_0u](https://twitter.com/Lo0_0u)

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

### Scope

The following smart contracts were in scope of the audit:

- [TestTask.sol]()

The following number of issues were found, categorized by their severity:

- Critical & High: 3 issues
- Medium: 3 issues
- Low: 2 issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [C-01] | Rewards are lost forever | Critical |
| [H-01] | Wrong Calculation in Collecting Reward First time | High     |
| [H-01] | Users lose stakes permanently if they call `emergencyWithdraw` | High     |
| [M-01] | Not all tokens `revert` in `transfer`   | Medium   |
| [M-02] | Missing check for if Stake Period Passed   | Medium   |
| [M-03] | `totalStaked` not updated after withdrawal   | Medium   |
| [L-01] | `fee` is not applied`      | Low      |
| [L-02] | Unused variables should be removed      | Low      |

# Detailed Findings

# [C-01] Rewards are lost forever

## Severity

**Impact:** High, as it is a value loss for users

**Likelihood:** High, as it is a common vulnerability and requires no preconditions

## Description

The `_stake` function doesn't work as intended and rewards would be stuck there forever. When the `stake` function is first called, it is required that `amount > 0`
```
    function stake(uint256 period, uint256 amount) external {
        require(amount > 0, "amount should be > 0");
```

This function then makes an internal call to the `_stake` function which implements the following code:

```
        if (amount == 0) {
            until = block.timestamp + (periods * _DAY * _DAYS_IN_MONT);
        } else {
            // @audit this gives a wrong value
            until = stakes[account].activeUntil;
        }

        _setStakeInfo(account, newAmount, periods, block.timestamp, until);
```

The issue here is that `until` would be set to 0 as `stakes[account].activeUntil` would be initialized as 0.

The `_rewardAmount` function would in turn be affected as `time` would always be 0, `periodPassed` would be 0 and therefore, `reward` would also be zero (0).

```
        //@audit-info this would always be true
        if (block.timestamp > stakeInfo.activeUntil) {
            time = stakeInfo.activeUntil;
        } else {
            time = block.timestamp;
        }
        // @audit Calculation is wrong when users claim reward the first time
        periodsPassed = (time - lastRewardClaims[account]) / _DAY;
        //Div(360) added because the reward percent is yearly reward.
        //1e8 multiplication and dividing added to mitigate rounding issues.
        reward =
            (stakeInfo.amount * rewardRate * periodsPassed * 1e8) /
            100 /
            _YEAR_IN_DAYS /
            1e8;
```

This would also affect the `_nextRewardDate` function as it would always return 0.

```
        // @audit-info this would always return 0 as stakeInfo.activeUntil = 0
        if (block.timestamp > stakeInfo.activeUntil) {
            return stakeInfo.activeUntil;
        }
```


## Recommendations

`until` should be properly updated with `until = block.timestamp + (periods * _DAY * _DAYS_IN_MONT);`

# [H-01] Wrong Calculation in Collecting Reward First time

## Severity

**Impact:** Medium, as it only happens on the first claim

**Likelihood:** High, as it is a common vulnerability and requires no preconditions 

## Description

In the `_rewardAmount` function, the `periodPassed` is calculated with this:

```
        // @audit Calculation is wrong when users claim reward the first time
        periodsPassed = (time - lastRewardClaims[account]) / _DAY;
```

If a user calls `_collectReward` for the first time, and before the end of the period, then, `time = block.timestamp;`. `block.timestamp` is the UNIX timestamp (the number of seconds since January 1970), and if `lastRewardClaims[account]` = 0, then the `periodPassed` would be the `block.timestamp / _DAY` which would be a value larger than intended.

## Recommendations

Add a condition to check if `lastRewardClaims[account]` = 0, and then use `stakes[account].startedAt` instead of `lastRewardClaims[account]`.

# [H-02] Users lose stakes permanently if they call `emergencyWithdraw`

## Severity

**Impact:** High, as it is a value loss for users

**Likelihood:** Medium

## Description

In the `emergencyWithdraw` function, the `stakes[msg.sender].amount` is updated to 0 before trying to withdraw.

```
        stakes[msg.sender].amount = 0;

        // @audit Loss of fund as: stakes[msg.sender].amount = 0
        _withdraw(msg.sender, stakes[msg.sender].amount);
    }
```

## Recommendations

Consider storing the value of `stakes[msg.sender].amount` before updating it to 0

```
    +    uint256 value = stakes[msg.sender].amount; 
        stakes[msg.sender].amount = 0;

    +    _withdraw(msg.sender, value);
        
        // @audit Loss of fund as: stakes[msg.sender].amount = 0
    -    _withdraw(msg.sender, stakes[msg.sender].amount);

```

# [M-01] Not all tokens `revert` in `transfer`

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `_withdraw` function, the return value of `stakingToken.transfer(account, _amount);` was not checked.

```
    function _withdraw(address account, uint256 _amount) private {
        _setStakeInfo(account, 0, 0, 0, 0);
        // @audit not all tokens revert with transfer
        stakingToken.transfer(account, _amount);
        emit Withdraw(account, _amount, false);
    }
```

## Recommendations

Consider checking the `return` value as not all tokens revert in `transfer` when they fail.

# [M-02] Missing check for if Stake Period Passed

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `withdraw` function, there is no check for if the stake period has elapsed. Therefore, users can withdraw at anytime, bypassing the `emergencyWithdraw()` function and not needing to pay the associated fee when withdrawing early.

```
    /**
     * @notice withdraw all tokens if the stake period has passed.
     */
    function withdraw() external {
        require(stakes[msg.sender].amount > 0, "no stake");
        _collectRewards(msg.sender, true);
        // @audit No check if stake period has passed
        _withdraw(msg.sender, stakes[msg.sender].amount);
    }
```

## Recommendations

Consider checking the period passed before allowing withdrawal.

# [M-03] `totalStaked` not updated after withdrawal

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `withdraw` function, the state variable `totalStaked` was not updated after a withdrawal.

```
    /**
     * @notice withdraw all tokens if the stake period has passed.
     */
     
    function withdraw() external {
        require(stakes[msg.sender].amount > 0, "no stake");
        _collectRewards(msg.sender, true);
        // @audit No check if stake period has passed
        _withdraw(msg.sender, stakes[msg.sender].amount);
        // @audit `totalStaked`not updated
    }
```

## Recommendations

Update `totalStaked` in the `withdraw()` function.

# [L-01] `fee` is not applied`

## Severity

**Impact:** Low

**Likelihood:** Medium

## Description

`earlyWithdrawalFee` is declared and set: `earlyWithdrawFee = 10;` but in the `emergencyWithdraw` function and `prolong` function, `fee` is not applied.

```
    /**
     * @notice withdraw all tokens before stake period is not passed.
     * Fee is applied and will be burned.
     */

    // @audit `earlyWithdrawalFee` is not applied.
    function emergencyWithdraw() external {
        require(stakes[msg.sender].amount > 0, "no stake");

        totalStaked -= stakes[msg.sender].amount;
        stakes[msg.sender].amount = 0;

        _withdraw(msg.sender, stakes[msg.sender].amount);
    }
```

```
    /**
     * @notice prolong stake
     * Fee is applied and will be burned.
     */
    // @audit fee is not applied here
    function prolong(uint256 period) external {
        StakeInfo storage stakeInfo = stakes[msg.sender];
        require(stakeInfo.amount > 0, "stake required");
        require(stakeInfo.activeUntil < block.timestamp, "still active");
        _collectRewards(msg.sender, true);
        _stake(msg.sender, period, 0);
    }
```

## Recommendations

Add the `earlyWithdrawalFee` in the implementation as stipulated.

# [L-02] Unused variables should be removed 

## Severity

**Impact:** Low

**Likelihood:** Low

## Description

`levelPeriods` and `_operators` were not used in the contract.
```
    mapping(uint256 => uint256[]) public levelPeriods; //@audit Not used
    mapping(address => uint256) public lastRewardClaims;
    mapping(address => bool) private _operators; // @audit Not used
```

## Recommendations

Consider removing unused functions.