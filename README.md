# Swap contract example using uniswap contract package 

유니스왑에서 제공해주는 컨트랙트 인터페이스와 코드를 사용한 간단한 스왑 컨트랙트 구현 예시.

물론 프로덕션 환경에서 이대로 사용하기엔 불안정한 면이 많다고 한다. 

서비스에 맞는 추가적인 정보가 필요할 것이고, 저장할 정보가 많다면 스토리지를 관리할 컨트랙트를 분리하는 것도 방법중에 하나라고 한다.

## Single swap contract

SingleSwap 컨트랙트는 Uniswap docs에서 가져온 Full example이다. 

DAI, WETH9, USDC 컨트랙트의 주소를 하드코딩으로 선언해둔 것을 볼 수 있다. 

```solidity
address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address public constant WETH9 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
```

그 다음으로 주목할 것은 아래의 `ISwapRouter`이다. 

```solidity
ISwapRouter public immutable swapRouter;

constructor(ISwapRouter _swapRouter) {
	swapRouter = _swapRouter;
}
```

사용할 인터페이스를 정의해둔 부분이다. 해당 인터페이스를 구현하는 컨트랙트를 받아서 주입해준다.

주요 로직은 두 가지로 나뉜다. 

1. swapExactInputSingle
2. swapExactOutputSingle

두 가지 함수에 대해서 살펴보자.

### swapExactInputSingle

```solidity
function swapExactInputSingle(uint256 amountIn)
	external
	returns (uint256 amountOut)
{
	TransferHelper.safeTransferFrom(
		DAI,
		msg.sender,
		address(this),
		amountIn
	);

	TransferHelper.safeApprove(DAI, address(swapRouter), amountIn);

	ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
		.ExactInputSingleParams({
			tokenIn: DAI,
			tokenOut: WETH9,
			fee: poolFee,
			recipient: msg.sender,
			deadline: block.timestamp,
			amountIn: amountIn,
			amountOutMinimum: 0,
			sqrtPriceLimitX96: 0
		});

	amountOut = swapRouter.exactInputSingle(params);
}
```

천천히 살펴보면, `uint256 amountIn`이 들어와서 `uint256 amountOut`이 나가는 것 뿐이다.

`TransferHelper`를 통해서 `DAI`를 `msg.sender`로부터 `address(this)`로 `amountIn` 만큼 옮긴다.

그리고 나서 TransferHelper를 통해서 `DAI` 컨트랙트로 `swapRouter`를 `amountIn`만큼 approve 해준다.

그 다음은 `ISwapRouter`를 이용해서 `exactInputSingle`을 사용하기 위한 param을 만들어 실행한다. 

들어온 token은 `DAI`, 나가는 token은 `WETH9`이 되고, fee는 컨트랙트에 선언된 `poolFee`이다.

`deadline`은 장기 보류 트랜잭션 및 급격한 가격 변동으로부터 보호하기 위해 스왑이 실패하는 `UNIX`시간이다. 

recipient(받는 사람)은 `msg.sender`에 해당하고, 들어온 양은 amountIn이다. 

`amountOutMinimum`이 0으로 설정되어 있는데, 프로덕션에서는 위험하다. 

실제 배포의 경우 이 값은 Uniswap SDK 혹은 온체인 price oracle을 통해 계산 과정을 거쳐야한다. 

`sqrtPriceLimitX96` 값을 0으로 설정하여 이 매개변수를 비활성화했다. 

프로덕션에서 이 값을 사용하여 스왑이 풀에 푸시할 가격 한도를 설정하여 가격 영향으로부터 보호하거나 다양한 가격 관련 메커니즘에서 논리를 설정하는 데 도움이 될 수 있다.

```solidity 
function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256 amountOut);
```

Interface 상에서는 위와 같이 표현되어 있다. 

해당 함수를 구현하는 컨트랙트를 생성자로 받았기 때문에 ISwapRouter를 구현한 컨트랙트에서 실행이 될 것이다. 

## Multi swap contract

SingleSwap 컨트랙트 역시 Uniswap docs에서 가져온 Full example이다. 

마찬가지로 DAI, WETH9, USDC 컨트랙트의 주소를 하드코딩으로 선언해둔 것을 볼 수 있다. 

`swapExactInputMultihop`과 `swapExactOutputMultihop`이 있다. 두 함수를 중점적으로 살펴보자.

```solidity
function swapExactInputMultihop(uint256 amountIn)
        external
        returns (uint256 amountOut)
    {
        TransferHelper.safeTransferFrom(
            DAI,
            msg.sender,
            address(this),
            amountIn
        );

        TransferHelper.safeApprove(DAI, address(swapRouter), amountIn);

        ISwapRouter.ExactInputParams memory params = ISwapRouter
            .ExactInputParams({
                path: abi.encodePacked(DAI, poolFee, USDC, poolFee, WETH9),
                recipient: msg.sender,
                deadline: block.timestamp,
                amountIn: amountIn,
                amountOutMinimum: 0
            });

        amountOut = swapRouter.exactInput(params);
    }
```
큰 골자는 `SingleSwap`과 일치한다. 

`ExactInputParams`의 `path`가 주목해서 볼만한데, 해당 인자는 `tokenAddress`, `Fee`, `tokenAddress` 를 역순으로 인코딩한 시퀀스이다. 

멀티홉 스왑 라우터는 이러한 변수가 있는 풀을 찾아서 풀 내에서 필요한 스왑을 순차적으로 실행한다.

## Providing liquidity 

유동성 제공에 대한 컨트랙트 역시 Uniswap Docs를 참고했다. 

```solidity
address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
```

마찬가지로 하드코딩된 DAI와 USDC.

프로덕션 환경에서는 입력 매개변수를 받아 트랜잭션별로 상호작용하는 풀과 토큰을 변경하면 될 것이다. 

```solidity
INonfungiblePositionManager public immutable nonfungiblePositionManager;
```

`immutable`로 선언한 `INonfungiblePositionManager`가 보인다. 

```solidity
struct Deposit {
	address owner;
	uint128 liquidity;
	address token0;
	address token1;
}
```

`Deposit` 구조체를 살펴보면 `owner address`와 `uint128`로 선언된 유동성, 그리고 `token0`과 `token1`의 주소가 들어가있다.

자잘한건 전부 넘기고, 함수들을 위주로 살펴보자.

```solidity
function onERC721Received(
	address operator,
	address,
	uint256 tokenId,
	bytes calldata
) external override returns (bytes4) {
	// get position information

	_createDeposit(operator, tokenId);

	return this.onERC721Received.selector;
}

function _createDeposit(address owner, uint256 tokenId) internal {
	(
		,
		,
		address token0,
		address token1,
		,
		,
		,
		uint128 liquidity,
		,
		,
		,

	) = nonfungiblePositionManager.positions(tokenId);

	// set the owner and data for position
	// operator is msg.sender
	deposits[tokenId] = Deposit({
		owner: owner,
		liquidity: liquidity,
		token0: token0,
		token1: token1
	});
}
```

당황스러운 코드지만 천천히 살펴보자.

`onERC721Received`는 단순히 컨트랙트가 ERC721 토큰을 보관하게 하도록 상속받은 함수이다. 

중요한 것은 `_createDeposit` 함수일 것이다.

`nonfungiblePositionManager.positions(tokenId)` 를 사용해서 필요한 정보를 받아온다. 

`address token0`, `address token1`, `uint128 liquidity`가 그에 해당한다. 

그리고 mapping 구조체에 `tokenId`를 key로 하는 `Deposit 구조체`를 삽입한다.

## Liquidity mining

## Flash swaps

## Governance proposal 