# 智能合约开发最佳实践-针对Solidity的建议
以下建议针对于Solidity，但对于开发其他语言的智能合约也可能具有指导意义。

## 使用`assert()`强制检查
断言失败时触发断言保护。例如，在token发行合约中，token与以太币的发行比率可以是固定的，你可以始终
使用`assert()`验证情况是否如此。断言保护通常应与其它技术结合使用，例如暂停合约和允许升级。

例如：

```solidity
contract Token{
    mapping(address => uint) public balanceOf;
    uint public totalSupply;
    
    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(address(this).balance >= totalSupply);
    }
}
```

请注意，上面不能用等号判断，因为可以在不通过`deposit()`函数的情况下将以太币强制发送给合约。

## 正确使用`assert(), require(), revert()`
> 函数`assert`和`require`可以用于检查条件，如果不满足条件则抛出异常。
> `assert`方法应仅用于测试内部错误和检查不变性
> `require`方法应用于确保满足输入或合约状态变量之类的有效条件，或用于验证从调用外部合约获得的返回值

```solidity
pragma solidity ^0.5.0;

contract Sharer {
    function sendHalf(address payable addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0, 'Event value required.');
        uint balanceBeforeTransfer = address(this).balance;
        (bool success, ) = addr.call.value(msg.value / 2)("");
        require(success);
        //assert用于内部错误检查 
        assert(address(this).balance == balanceBeforeTransfer - msg.value / 2);
        return address(this).balance;
    }
}
```

## 修饰符modifier仅用于条件检查
修饰符内部的代码通常在函数主题之前执行，因此任何状态更改或外部调用都将违反checks-effects-interactions模式。
因此，由于修饰符的代码可能与函数声明相去甚远，因此开发人员可能也不会注意到这些语句，例如，修饰符中的
外部调用可能导致重入攻击

```solidity
contract Registry {
    address owner;
    
    function isVoter(address _addr) external returns(bool) {
        // code
        return true;
    }
}

contract Election {
    Registry registry;
    
    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }
    
    function vote() isEligible(msg.sender) public {
        // code
    }
}
```
在这种情况下，`Registry`合约可以通过在`isVoter()`中调用`Election.vote()`来进行重入攻击。

##小心整数除法舍入
所有整数除法均四舍五入为最接近的整数。如果需要更高的精度，请考虑使用乘数，或者同事存储分子和分母。
(未来，Solidity将会有一个`fixed-point`类型，这将使此操作更容易。)
```solidity
// bad
uint x = 5 / 2; // 结果是2，所有整数除法均向下舍入到最接近的整数。
```
使用乘数可以防止舍入，将来使用x时需要考虑此乘数。
```solidity
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```
存储分子和分母意味着你可以在链下计算分子/分母的结果。
```solidity
//good
uint numerator = 5;
uint denominator = 2;
```

##注意抽象合约和接口之间的权衡
接口和抽象合约都为智能合约提供了一种可自定义和可重复使用的方法。在`Solidity 0.4.11`中引入的接口
和抽象合约类似，但是不能实现任何功能。接口还具有局限性，例如无法访问存储或无法从其它接口继承，这通常
使抽象合约更加实用。虽然，接口对于在实现之前设计合约肯定是有用的。此外，重要的是要记住，如果合约从抽
象合约继承，那么它必须通过覆盖实现所有未实现的功能，否则它也将是抽象的。

## 回调方法
**保持回调方法简单** 

向合约发送不带参数的消息(或没有函数匹配)时调用回调函数，从`send()`或`transfer()`调用时只能使
用2300gas，如果你希望能够从`send()`或`transfer()`接收以太币，则在回调函数中可以做的最大的事情
就是记录一个事件，如果需要更多的gas，请使用适当的函数。

```solidity
//bad
function() payable { balances[msg.sender] += msg.value; }

//good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender);}
```

**检查回调方法中的data长度** 

由于回调方法不仅会调用纯ether交易(没有data)，还会在没有其它方法匹配的时候调用，如果回调方法仅用于
记录接收到的以太币，则应检查数据是否为空。否则，调用者将不会注意到你的合约使用不正确，并且调用了不存
在的函数。
```solidity
//bad
function() payable { emit LogDepositReceived(msg.sender); }

//good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

## 明确标记payable方法和状态变量
从Solidity `0.4.0` 开始，每个接收以太币的函数都必须使用`payable`修饰符，否则，如果交易的
`msg.value > 0` 将被`revert`

> 可能不那么明显, `payable`修饰符仅适用于来自外部合约的调用。如果我在同一合约的`payable`方法中
> 调用了`non-payable`方法，即使设置了`msg.value`，`non-payable`方法也不会失败。

## 明确标记函数和状态变量的可见性
明确标记函数和状态变量的可见性。函数可以将功能指定为`external, public, internal, private`，
请理解它们之间的区别，例如，`external`可能足以替代`public`，对于状态变量，`external`是不可用的。
明确标记可见性将使你更容易发现关于谁可以调用函数或访问变量的错误假设。
- `External` 函数是合约接口的一部分。外部函数`f`不能被内部调用(i.e. `f()`不起作用，但是`this.f()`有效)。
当接收大量数据时，使用外部函数更有效。
- `Public` 函数是合约接口的一部分。可以在内部或通过`call`进行调用。对于`public`的状态变量，将自动
生成一个`getter`方法。
- `Internal` 函数和状态变量只能内部访问，而不能使用它。
- `Private` 函数和状态变量仅对定义的合约可见，而在派生合约中不可见。**注意：合约内的所有内容，对于
区块链外部的所有观察者都是可见的，甚至是似有变量（Private）**

```solidity
//bad
uint x; // 默认是internal的，但是应该明确
function buy() { // 默认是public的
    // public code
}

//good
uint private y;
function buy() external {
    // 只能在外部调用或者使用this.buy()
}

function utility() public {
    // 可以在内部和外部调用：更改此代码需要考虑两种调用的可能情况
}

function internalAction() internal {
    // internal code
}
```

## 将编译指示锁定到特定的编译器版本
合约应使用经过最多测试的相同编译器版本和标志进行部署。锁定pragma可以确保不会使用例如最新的编译器
意外的部署合约，而最新的编译器可能会更容易发现未发现的错误。合约也可以由其他人部署，并且pragma指示
原始作者打算使用的编译器版本。
```solidity
//bad
pragma solidity ^0.4.4;

//good
pragma solidity 0.4.4;
```

注意：浮动编译指示版本(即`^0.4.25`)可以在`0.4.26-nightly.2018.9.25`上正常编译，但是每晚构建
不应用作编译生产代码。

>当合约打算供其他人使用时，可以允许pragma语句浮动，例如库或者EthPM软件包中的合约。否则，开发人员
>将需要手动更新编译器以便本地编译

## 使用事件监视合约活动
部署合约后，有一种监视合约的方法可能会很有用。实现此目的的一种方法是查看合约的所有交易，但是由于
合约之间的消息调用未记录在区块链中，因此这可能是不够的。此外，它仅显示输入参数，而不显示对该状态的
实际修改。事件也可以用于除法用户界面中的功能。
```solidity
contract Charity {
    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5%给charity
        charity.donate.value(msg.value / 20)();
    }
}
```
在这里，Game合约将对`charity.donate`进行内部调用。该交易不会出现在`charity`的外部交易列表中，而
只会在内部交易中显示。

事件是记录合约中发生的事情的便捷的方法。发出的事件与其它合约数据一起留在区块链中，可供以后审核，这是
对以上示例的改进，使用事件提供了慈善捐赠的历史记录。

```solidity
contract Charity {
    // 定义事件
    event LogDonate(uint _amount);

    mapping(address => uint) balances;
    
    function donate() payable public {
        balances[msg.sender] += msg.value;
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        charity.donate.value(msg.value / 20)();
    }
}
```
在这里，所有通过`Charity`合约的交易，无论直接与否，都将与捐赠金额一起显示在该合约的事件列表中。

>**更喜欢新的Solidity结构**。更喜欢`selfdestruct`(好过`suicide`)和`keccak256`(好过`sha3`).
>像这样的模式`require(msg.sender.value(1 ether))`也可以简单的使用`transfer()`，`msg.sender.transfer(1 ether)`.

## 请注意`内置`可能被遮盖
目前可以在Solidity中隐藏内置全局变量。这允许合约覆盖诸如`msg`和`revert()`之类的内置功能。尽管
这样做是有意的，但它可能会误导合约的用户，使他们了解合约的真实行为。
```solidity
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```
合约用户和审核员应该了解他们打算使用的任何应用程序的完整智能合约源码