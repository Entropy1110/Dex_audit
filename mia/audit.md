## medium - 토큰 유동성 풀의 AMM이 손상되어 비정상적인 가격 변동이 발생할 수 있습니다.

### 설명

./src/Dex.sol : 67

**addLiquidity(uint256,uint256,uint256) external override refresh returns (uint256)**

```solidity
function addLiquidity(uint256 amountX, uint256 amountY, uint256 minLPReturn)
        external
        override
        refresh
        returns (uint256 lpAmount)
    {
        // 유동성 풀의 비율과 일치하는지 확인합니다.
        require(balanceX / amountX == balanceY / amountY);
```

유동성 풀의 비율에 맞는 amountX, amountY 인자 값을 검사하는 logic에 오류가 있습니다.

유동성 풀의 비율에 일치하지 않는 amountX, amountY가 require문을 통과할 수 있습니다.

`balanceX = 10ether, balanceY = 20ether`

`amountX = 100ether, amountY = 30ether` 인 경우, 

`require(balanceX / amountX == balanceY / amountY);` 조건에서

`(0 == 0)`으로 다른 비율의 X, Y 토큰으로 통과할 수 있습니다.

### 파급력

Medium

1. Contract의 balancX/Y 보다 큰 값을 유동성 풀에 추가할 경우, 가격 변동이 발생하여 공격자가 swap 가격을 결정할 수 있습니다. 

```solidity
contract DexScript is Test {
    IDex public dex;
    IERC20 tokenX;
    IERC20 tokenY;

    function setUp() public {
        tokenX = new CustomERC20("X");
        tokenY = new CustomERC20("Y");
        dex = new Dex(address(tokenX), address(tokenY));
        tokenX.approve(address(dex), type(uint256).max);
        tokenY.approve(address(dex), type(uint256).max);

        dex.addLiquidity(100 ether, 100 ether, 0);
    }

    function test_exploit() public {
        
        uint LPReturn1 = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LPReturn1: ", LPReturn1);

        uint swap1 = dex.swap(10 ether, 0, 0);
        console.log("swap1: ", swap1);

        
    }

    function test_exploit2() public {
        uint LPReturn2 = dex.addLiquidity(102 ether, 100000 ether, 0);
        console.log("LPReturn2: ", LPReturn2);

        uint swap2 = dex.swap(10 ether, 0, 0);
        console.log("swap2: ", swap2);
    }
}
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dcc66554-0f51-432b-b52f-51edb25200cb/307f4e78-ba24-4fa7-b57c-e2a80dd662f4/image.png)

### 해결 방안

1. addLiquidity 함수의 인자가 아닌 실제로 추가되어야할 토큰의 양을 현재 유동성 풀의 비율에 대해 계산하여 transfer하는 방법을 사용할 수 있습니다.
    1. ./src/Dex.sol : 67
    amountX, amountY 중 실제로 유동성 풀에 추가될 수 있는 양을 계산 → aX, aY
    2. ./src/Dex.sol : 86-87
    `tokenX.transferFrom(msg.sender, address(this), aX);`
    `tokenY.transferFrom(msg.sender, address(this), aY);`
2. reserve 양을 변수로 사용하고, skim 함수를 implement하여 필요한 경우 유동성 풀의 비율을 맞춰줄 수 있습니다.
    
    ```solidity
    function skim(address to) external lock {
            address _token0 = token0; // gas savings
            address _token1 = token1; // gas savings
            _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
            _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
        }
    ```
