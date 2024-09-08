# jacqueline - DEX_solidity Security Audit

# Table of Contents

# Audit Overview

### Name

- jacqueline - DEX_solidity Security Audit

### Git Repository

- [https://github.com/je1att0/DEX_solidity](https://github.com/je1att0/DEX_solidity)

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
| Critical | 1 | - `DEX-jacqueline-001` |
| High |  |  |
| Medium | 1 | - `DEX-jacqueline-002` |
| Low |  |  |
| Informational |  |  |

# Findings

## Summary

| # | ID | Title | Severity |
| --- | --- | --- | --- |
| 1 | `DEX-jacqueline-001` | Not checking input amount of LP token may lead to the theft of all the tokens in the pool | Critical |
| 2 | `DEX-jacqueline-002` | Minting and approving maximum amount of LP token may lead to potential risk. | Medium |

## #1 `DEX-jacqueline-001` Not checking input amount of LP token may lead to the theft of all the tokens in the pool

### Description

- src/Dex.sol: 49-63 [removeLiquidity()]

```jsx
function removeLiquidity(uint _lpReturn, uint _minAmountX, uint _minAmountY) public returns (uint _tx, uint _ty) {
        uint poolAmountX = tokenX.balanceOf(address(this));
        uint poolAmountY = tokenY.balanceOf(address(this));
        _tx = poolAmountX*_lpReturn/totalLP;
        _ty = poolAmountY*_lpReturn/totalLP;
        require(_tx >= _minAmountX && _ty >= _minAmountY, "Insufficient minimum amount");
        require(_lpReturn <= poolAmountX + poolAmountY, "Insufficient LP return");

        tokenX.transfer(msg.sender, _tx);
        tokenY.transfer(msg.sender, _ty);
        totalLP -= _lpReturn;

        return (_tx, _ty);

    }
```

본 컨트랙트에서는 유저가 removeLiquidity를 호출하여 제공했던 유동성을 제거하려고 할 때, 유저에게 주었던 LP 토큰을 돌려받아서 burn하는 코드가 없다. 따라서 유저는 LP 토큰을 실제로 돌려주지 않아도 removeLiquidity를 호출할 때 _lpReturn에 0보다 큰 숫자를 인자로 주기만 하면 임의로 토큰 X와 Y를 transfer 받을 수 있다. 따라서 기존에 유동성을 공급했던 사용자는 다시 X, Y 토큰을 돌려받게 되면서 LP 토큰이 추가로 자산에 생기게 되어 이를 외부 거래소에서 거래하여 추가적인 이익을 취할 수 있다. 또한, 유동성을 기존에 공급하지 않았더라도 임의로 removeLiquidity를 호출하여 바로 토큰 X, Y를 transfer 받을 수 있다

### Impact

- Critical

공격자가 removeLiquidity를 호출하기만 하면 DEX 풀에 있는 모든 X, Y 유동성을 제공할 수 있다.

### Recommendation

유저가 removeLiquidity를 호출할 때 인자로 주는 _lpReturn  양이 실제로 유저에게 제공된 LP 토큰의 양 이하인지 체크하는 로직을 추가하고, LP 토큰을 transfer 받은 후 이를 burn 하는 코드를 추가한다.

## #2 `DEX-jacqueline-002` Minting and approving maximum amount of LP token may lead to potential risk

### Description

- src/Dex.sol: 6-10

```jsx
contract CustomERC20 is ERC20 {
    constructor(string memory tokenName) ERC20(tokenName, tokenName) {
        _mint(msg.sender, type(uint).max);
    }
}
```

- src/Dex.sol: 18-23

```jsx
    constructor (address _tokenX, address _tokenY) {
        tokenX = ERC20(_tokenX);
        tokenY = ERC20(_tokenY);
        tokenLP = new CustomERC20("LP");
        tokenLP.approve(address(this), type(uint).max);
    }
```

본 컨트랙트에서는 제공된 X, Y 유동성에 따라 LP 토큰을 발행하고 유동성 제공자에게 transfer 해주는 것이 아니라, 초기에 max LP 토큰을 발행한다. 시스템 내에서 너무 많은 토큰이 발행되어 경제적인 문제를 일으킬 수 있다. 특히 LP 토큰의 총 공급량을 제대로 관리하지 않는다면 심각한 인플레이션을 초래할 수 있으므로 제한해야 한다. 

### Impact

- Medium

필요할 때마다 필요한 양만큼 발행하고 approve 해주지 않고 무제한 허가를 하는 것은 잠재적인 보안 위험에 노출될 위험이 크다. 공격자가 어떠한 방법으로 해당 컨트랙트의 주권을 가지고 오면, 이 컨트랙트에 존재하는 모든 LP 풀은 직접적인 위험에 노출된다.

### Recommendation

유동성 제공자가 제공한 X, Y 비율에 따라 LP 토큰을 발행한다.