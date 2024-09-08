# ella - DEX_solidity Security Audit

# Table of Contents

# Audit Overview

### Name

- ella - DEX_solidity Security Audit

### Git Repository

- [https://github.com/skskgus/Dex_solidity](https://github.com/skskgus/Dex_solidity)

### Severity Categories

- Critical
    
    The attack cost is low (not requiring much time or effort to succeed in the
    actual attack), and the vulnerability causes a high-impact issue. (e.g., Effect on
    service availability, Attacker taking financial gain)
    
- High
    
    An attacker can succeed in an attack which clearly causes problems in the
    service’s operation. Even when the attack cost is high, the severity of the issue
    is considered “high” if the impact of the attack is remarkably high.
    
- Medium
    
    An attacker may perform an unintended action in the service, and the action
    may impact service operation. However, there are some restrictions for the
    actual attack to succeed.
    
- Low
    
    An attacker can perform an unintended action in the service, but the action
    does not cause significant impact or the success rate of the attack is
    remarkably low.
    
- Informational
    
    Any informational findings that do not directly impact the user or the protocol.
    

### Finding Breakdown by Severity

| Category | Count | Findings |
| --- | --- | --- |
| Critical | 1 | `DEX-ella-001` |
| High | 1 | `DEX-ella-002` |
| Medium |  |  |
| Low |  |  |
| Informational |  |  |

# Findings

## Summary

| # | ID | Title | Severity |
| --- | --- | --- | --- |
| 1 | `DEX-ella-001` | Not checking the LP token amount of users in removeLiquidity may lead to the theft of all the tokens in the pool | Critical |
| 2 | `DEX-ella-002` | Not setting minimum liquidity may lead to unstable liquidity of the pool | High |

## #1 `DEX-ella-001` Not checking the LP token amount of users in removeLiquidity may lead to the theft of all the tokens in the pool

### Description

- src/Dex.sol: 90-109 [removeLiquidity()]

```jsx
    function removeLiquidity(uint256 shares, uint256 minAmount0, uint256 minAmount1) external returns (uint256 amount0, uint256 amount1) {
        uint256 _reserve0 = reserve0;
        uint256 _reserve1 = reserve1;
        uint256 _totalSupply = totalSupply;

        amount0 = (shares * _reserve0) / _totalSupply;
        amount1 = (shares * _reserve1) / _totalSupply;

        require(amount0 >= minAmount0, "Insufficient amount0");
        require(amount1 >= minAmount1, "Insufficient amount1");

        _burn(msg.sender, shares);

        _update();

        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);

        _update();
    }
```

본 컨트랙트에서는 removeLiquidity 호출 시 유저가 호출 인자로 준 shares만큼의 LP token을 실제로 가지고 있는지 확인하지 않는다. 공격자는 이를 이용하여 0보다 큰 임의의 값을 shares 인자로 주고 토큰 X, Y를 transfer 받을 수 있다.

### Impact

- Critical

공격자가 removeLiquidity를 호출하기만 하면 토큰 X, Y에 대한 유동성을 모두 제거해버릴 수 있다.

### Recommendation

addLiquidity 시 유저에게 발행된 LP 토큰을 스토리지에 기록하여 유저가 유동성 제거를 요청하며 LP token 양을 인자로 줄 때 유효한지 체크하는 로직을 추가한다.

## #2 `DEX-ella-002` Not setting minimum liquidity may lead to unstable liquidity of the pool

### Description

- src/Dex.sol: 66-67 [addLiquidity()]

```jsx
        if (totalSupply == 0) {
            shares = _sqrt(_amount0 * _amount1);
```

본 컨트랙트에서는 첫 유동성 제공자에게 위와 같이 LP token을 지급한다. 만약 첫 유동성 제공자가 매우 큰 양의 유동성을 제공해서, 전체 풀에 큰 지분을 갖고 있다면, 그 유동성 제공자가 유동성을 제거하려고 할 때, DEX 풀의 유동성이 급격하게 줄어들 수 있다. 그럴 경우 swap 등의 기능이 원활하게 이루어지지 못한다.

### Impact

- High

첫 유동성 제공자가 매우 큰 유동성을 제공하고, 제거하는 경우 전체 풀의 유동성을 크게 감소시킨다. 하지만 시간이 지나 다른 유동성 제공자들이 추가되면, 첫 유동성 제공자가 지니는 지분이 상대적으로 작아져 위험이 줄어든다.

### Recommendation

totalSupply가 0일 때, 즉, 첫 유동성 제공자에게 LP 토큰을 제공할 때, minimum liquidity 를 정하여 그 만큼의 값은 빼고 준다. 그럴 경우 항상 minimum liquidity 양 만큼의 유동성은 유지할 수 있다.