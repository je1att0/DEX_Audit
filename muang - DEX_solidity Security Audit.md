# muang - DEX_solidity Security Audit

# Table of Contents

# Audit Overview

### Name

- muang - DEX_solidity Security Audit

### Git Repository

- [https://github.com/GODMuang/DEX_solidity](https://github.com/GODMuang/DEX_solidity)

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
| Critical | 1 | - `DEX-muang-001` |
| High | 2 | - `DEX-muang-002`
- `DEX-muang-003` |
| Medium |  |  |
| Low |  |  |
| Informational |  |  |

# Findings

## Summary

| # | ID | Title | Severity |
| --- | --- | --- | --- |
| 1 | `DEX-muang-001` | addLiquidity() not considering ratio may lead to arbitrage and Impermanent Loss to liquidity providers | Critical |
| 2 | `DEX-muang-002` | Typecasting uint256 to uint112 in update() may lead to integer overflow | High |
| 3 | `DEX-muang-003` | Calculating amount out by dividing in swap() may lead to the theft of all the tokens in the pool  | High |

## #1 `DEX-muang-001` addLiquidity() not considering ratio may lead to arbitrage and Impermanent Loss to liquidity providers

### Description

- src/Dex.sol: 34-35 [addLiquidity()]

```jsx
        IERC20(tokenX).transferFrom(msg.sender, address(this), amountX);
        IERC20(tokenY).transferFrom(msg.sender, address(this), amountY);
```

본 컨트랙트의 addLiquidity 함수는 유동성 제공자가 X, Y 페어에 대해 유동성을 제공하고자 하는 만큼을 amountX, amountY 인자로 받고, 특정 조건들을 만족하면 그 값만큼을 그대로 풀에 추가한다. DEX가 풀 내 자산의 비율을 고려하지 않고 유동성 제공자가 제공한 X, Y 페어에 대한 공급을 전부 받아버리면 비율 대비 공급이 많아진 쪽의 자산 가치가 떨어져 아비트리지가 발생한다. 이럴 경우, 유동성 공급자는 비영구적 손실이 발생하고, 악의적인 차익 거래자들은 급격하게 가격 왜곡을 발생시켜 이익을 취하게 된다.

### Impact

- Critical

단순히 페어 토큰의 비율을 다르게 공급하여 풀 내 자산 비율이 달라지면 유동성 공급자에게 비영구적 손실을 유발하고, 공격자가는 다계정을 이용하여 DEX의 자산을 빠르게 고갈시키고 차익 거래를 통해 이익을 취할 수 있다. 유동성 공급자 입장에서는 손실 위험이 크므로 본 DEX에 유동성을 제공하지 않을 가능성이 높아지고, 유동성이 부족해지면 DEX의 기능이 약화되고 거래가 어려워진다.

### Recommendation

인자로 받은 amountX, amountY만큼을 그대로 풀에 추가하기 전에, 풀 내 자산의 비율에 비해 더 많이 들어오는 쪽의 양을 더 적게 들어오는 쪽의 양에 맞추어 풀에 추가한다.

## #2 `DEX-muang-002` Typecasting uint256 to uint112 in update() may lead to integer overflow

### Description

- src/Dex.sol: 10-11

```jsx
    uint112 private reserveX; 
    uint112 private reserveY;
```

- src/Dex.sol: 106-110 [update()]

```jsx
    function update(uint256 _balanceX, uint256 _balanceY) private {
    
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
```

처음에 reserveX와 reserveY는 uint112형의 전역변수로 선언되어 있다. 그런데 update() 함수에서 인자로 받아오는 _balanceX, _balanceY는 uint256형이라 이 값들을 각각 uint112형인 reserveX와 reserveY에 저장하는 과정에서 typecasting이 발생한다. uint112가 담을 수 있는 최대값은 5,192,296,858,534,827이므로 _balanceX 또는 _balanceY가 그 값보다 클 경우 integer overflow가 발생한다.

유동성 제공자가 유동성을 공급하면 addLiquidity() 함수에서 mint() 함수를 호출하여 LP 토큰을 유동성 제공자에게 보낸 후, update() 함수를 호출하여 유동성 공급 후 상태를 스토리지에 반영한다. 이 때, 악의적인 공격자가 유동성 공급을 한 후 풀의 가치가 5,192,296,858,534,827를 초과하도록 유동성을 공급하면 update() 함수 진행 시 overflow가 발생해 실제 풀의 자산 가치보다 적은 양이 기록에 반영될 수 있다.

### Impact

- High

공격에 성공할 경우, DEX 풀의 유동성을 감소시킬 수 있으므로 심각성이 높으나, 5,192,296,858,534,827를 초과하도록 유동성을 공급하는 것은 현실적으로 쉽지 않다.

### Recommendation

```jsx
require(_balanceX <= uint112(-1) && _balanceY <= uint112(-1), 'UniswapV2: OVERFLOW');
```

UniswapV2 코드와 같이 _balanceX 와 _balanceY가 uint112의 최댓값을 넘지 않도록 확인하는 조건문이 필요하다.

## #3 `DEX-muang-003` Calculating amount out by dividing in swap() may lead to the theft of all the tokens in the pool

### Description

- src/Dex.sol: 125, 133 [swap()]

```jsx
function swap(uint256 amountXIn, uint256 amountYIn, uint256 minTokenOut) external returns (uint)  {
        require((!(amountXIn == 0 && amountYIn == 0)) && !(amountXIn != 0 && amountYIn !=0), "INSUFFICIENT_SWAP_PARAMETER :((");
        (uint256 _reserveX, uint256 _reserveY) = getReserves();

        if(amountYIn == 0){
            uint amountYOut = _reserveY - ( _reserveX * _reserveY ) / ( _reserveX+amountXIn );
            uint amountYOutWithFee = ( amountYOut * 999 ) / 1000;
            tokenX.transferFrom(msg.sender, address(this), amountXIn);
            tokenY.transfer(msg.sender, amountYOutWithFee);
            require(amountYOut >= minTokenOut,"test");
            return amountYOutWithFee;
        
        }else{
            uint amountXOut = _reserveX - ( _reserveY * _reserveX ) / ( _reserveY+amountYIn );
            uint amountXOutWithFee = ( amountXOut * 999 ) / 1000;
            tokenY.transferFrom(msg.sender, address(this), amountYIn);
            tokenX.transfer(msg.sender, amountXOutWithFee);
            require(amountXOut >= minTokenOut);
            return amountXOutWithFee;
        }
    }
```

토큰 X를 토큰 Y로 스왑하려고 할 때, 전송해줄 Y의 양은  `uint amountYOut = _reserveY - ( _reserveX * _reserveY ) / ( _reserveX+amountXIn );` 이 식을 통해 계산된다. 솔리디티는 소수점 계산을 지원하지 않으므로 나눗셈 계산을 할 때 나누어 떨어지지 않으면 나머지를 버린다. 만약 악의적인 공격자 엄청나게 큰 양의 토큰 X를 넣어 _reserveX + amounXIn 이 _reserveX * _reserveY 보다 커지면 나누기 한 값은 0이 되어 전송해줄 Y의 양이 _reserveY, 즉, 풀 내에 있는 Y의 양과 같아져 Y의 모든 유동성을 출금할 수 있다. 
Y를 X로 스왑하려고 할 때도 동일한 문제가 발생한다.

### Impact

- High

공격에 성공할 경우, DEX 풀의 유동성을 감소시킬 수 있으므로 심각성이 높으나, 풀 내에 있는 X와 Y를 곱한 값보다 크도록 X를 넣는 것이 현실적으로 쉽지 않다.

### Recommendation

토큰을 넣은 후 풀의 잔액이 현재 두 풀 자산의 곱보다 작은 것을 확인하는 조건문을 추가한다.