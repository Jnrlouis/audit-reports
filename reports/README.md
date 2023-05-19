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

**_review commit hash_ - [4e527c16fe6b1524e9a8146ae122efc6](https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6)**

### Scope

The following smart contracts were in scope of the audit:

- [TestTask.sol](https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6)

The following number of issues were found, categorized by their severity:

- Critical & High: 3 issues
- Medium: 1 issues
- Low: 1 issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [C-01] | Rewards are lost forever | Critical |
| [H-01] | Wrong Calculation in Collecting Reward First time | High     |
| [H-01] | Users lose stakes permanently if they call `emergencyWithdraw` | High     |
| [M-01] | Not all tokens `revert` in `transfer`   | Medium   |
| [L-01] | `fee` is not applied`      | Low      |

# Detailed Findings

# [C-01] {Rewards are lost forever}

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

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L315-L321

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
https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L273-L285

This would also affect the `_nextRewardDate` function as it would always return 0.

```
        // @audit-info this would always return 0 as stakeInfo.activeUntil = 0
        if (block.timestamp > stakeInfo.activeUntil) {
            return stakeInfo.activeUntil;
        }
```

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L288-L295


## Recommendations

`until` should be properly updated with `until = block.timestamp + (periods * _DAY * _DAYS_IN_MONT);`

# [H-01] {Wrong Calculation in Collecting Reward First time}

## Severity

**Impact:** Medium, as it only happens on the first claim

**Likelihood:** High, as it is a common vulnerability and requires no preconditions 

## Description

In the `_rewardAmount` function, the `periodPassed` is calculated with this:

```
        // @audit Calculation is wrong when users claim reward the first time
        periodsPassed = (time - lastRewardClaims[account]) / _DAY;
```

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L273-L285

If a user tries calls `_collectReward` for the first time, and before the end of the period, then, `time = block.timestamp;`. `block.timestamp` is the UNIX timestamp (the number of seconds since January 1970), and if `lastRewardClaims[account]` = 0, then the `periodPassed` would be the `block.timestamp / _DAY` which would be a value larger than intended.

## Recommendations

Add a condition to check if `lastRewardClaims[account]` = 0, and then use `stakes[account].startedAt` instead of `lastRewardClaims[account]`.

# [H-02] {Users lose stakes permanently if they call `emergencyWithdraw`}

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

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L218-L220


## Recommendations

Consider storing the value of `stakes[msg.sender].amount` before updating it to 0

```
    +    uint256 value = stakes[msg.sender].amount; 
        stakes[msg.sender].amount = 0;

    +    _withdraw(msg.sender, value);
        
        // @audit Loss of fund as: stakes[msg.sender].amount = 0
    -    _withdraw(msg.sender, stakes[msg.sender].amount);

```

# [M-01] {Not all tokens `revert` in `transfer`}

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

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L326-L330


## Recommendations

Consider checking the `return` value as not all tokens revert in `transfer` when they fail.

# [L-01] {`fee` is not applied`}

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

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L210-L221

https://gist.github.com/EugeneBezuglyi/4e527c16fe6b1524e9a8146ae122efc6#file-testtask-sol-L223-L233


## Recommendations

Add the `earlyWithdrawalFee` in the implementation as stipulated.