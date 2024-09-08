# kenny - DEX_solidity Security Audit

# Table of Contents

# Audit Overview

### Name

- kenny - DEX_solidity Security Audit

### Git Repository

- [https://github.com/55hnnn/DEX_solidity](https://github.com/55hnnn/DEX_solidity)

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
| Critical | 1 | - `DEX-kenny-001` |
| High |  |  |
| Medium |  |  |
| Low |  |  |
| Informational |  |  |

# Findings

## Summary

| # | ID | Title | Severity |
| --- | --- | --- | --- |
| 1 | `DEX-kenny-001` | LP token not minted | Critical |

## #1 `DEX-kenny-001` LP token not minted

### Description

- src/Dex.sol: 34-36 [addLiquidity()]

```jsx
        // 유동성 토큰 발행
        liquidity[msg.sender] += liquidityMinted;
        totalLiquidity += liquidityMinted;
```

본 컨트랙트에서는 유저가 X, Y 유동성을 제공하면, LP 토큰을 직접 발행하고 transfer 해주는 것이 아니라 전역 변수 상에서 값만 tracking 하고 있다. LP 토큰이 발급되지 않으면 유저가 DEX에 기여한 정도를 증명하는 수단이 없고 그에 대한 보상도 없는 것이므로 유저 입장에서 이 DEX에 유동성을 공급할 이유가 없어 사용을 안하게 될 것이다. 유동성 공급이 부족해지면 DEX로서 원활하게 기능하지 못한다.

### Impact

- Critical

DEX에서 중요한 부분 중 하나인 유동성 공급이 줄어들 수 있으므로 위험하다. 

### Recommendation

유동성 공급에 대한 보상 또는 증명의 수단인 LP 토큰을 발행하고 transfer하는 코드를 추가한다.