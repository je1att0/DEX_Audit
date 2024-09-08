# damon - DEX_solidity Security Audit

# Table of Contents

# Audit Overview

### Name

- damon - DEX_solidity Security Audit

### Git Repository

- [https://github.com/gloomydumber/DEX_solidity](https://github.com/gloomydumber/DEX_solidity)

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
| Critical | 2 | - `DEX-damon-001`
- `DEX-damon-002` |
| High |  |  |
| Medium |  |  |
| Low |  |  |
| Informational |  |  |

# Findings

## Summary

| # | ID | Title | Severity |
| --- | --- | --- | --- |
| 1 | `DEX-damon-001` | Setting update() function to public may lead to the theft of all the tokens in the pool | Critical |
| 2 | `DEX-damon-002` | Liquidity to be burnt as a parameter of burn() may lead to the burnt of all the LP tokens in the pool  | Critical |

## #1 `DEX-damon-001` Setting update() function to public may lead to the theft of all the tokens in the pool

### Description

- src/Dex.sol: 138-142 [update()]

```jsx
    function update(uint256 _balanceX, uint256 _balanceY) public {
        require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
```

유동성 공급, swap 등으로 인해 풀 내 자산 양에 변동이 생겼을 때 update() 함수를 호출하여 스토리지 기록에 반영한다. 그런데 이 함수가 public으로 선언되어 있어 공격자가 풀 내 자산을 임의의 값으로 변경할 수 있다.

### Impact

- Critical

공격자가 이 함수를 호출하기만 하면 DEX의 유동성을 0으로 만들어 버릴 수 있다.

### Recommendation

update 함수를 private으로 변경한다.

## #2 `DEX-damon-002` Liquidity to be burnt as a parameter of burn() may lead to the burnt of all the LP tokens in the pool

### Description

- src/Dex.sol: 114-131 [burn()]

```jsx
    function burn(uint256 liquidity) external returns (uint256 _amountX, uint256 _amountY) {
        uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
        uint256 balanceY = IERC20(tokenY).balanceOf(address(this));
        uint256 totalSupply = totalSupply();

        _amountX = liquidity * balanceX / totalSupply;
        _amountY = liquidity * balanceY / totalSupply;

        require(_amountX > 0 && _amountY > 0, "INSUFFICENT_LIQUIDITY_BURNED");
        _burn(address(this), liquidity);
        safeTransfer(tokenX, msg.sender, _amountX);
        safeTransfer(tokenY, msg.sender, _amountY);

        balanceX = IERC20(tokenX).balanceOf(address(this));
        balanceY = IERC20(tokenY).balanceOf(address(this));

        update(balanceX, balanceY);
    }
```

removeLiquidity가 호출되면, 유저가 받은 LP 토큰을 burn 한다. 이 때, 현재 이 코드에서 burn 함수는 external로 선언되었으며, burn할 LP 토큰의 양 liquidity를 balanceOf 등으로 받아오는 것이 아니라 함수 호출 인자로 받는다. 따라서 공격자는 liquidity 인자로 0보다 큰 숫자를 주고 이 함수를 호출하면 이 풀 내에 있는 LP 토큰을 임의로 burn 할 수 있고, tokenX와 tokenY를 transfer 받을 수 있다.

### Impact

- Critical

공격자가 외부에서 0보다 큰 숫자를 인자로 주며 burn 함수를 호출하기만 하면 LP 토큰을 모두 제거할 수도 있고, 토큰 X와 Y를 제공받을 수 있다.

### Recommendation

burn 함수에서 burn 할 LP 토큰의 양을 함수 인자로 받지 말고 balanceOf(address(this)) 등으로 받아온다.