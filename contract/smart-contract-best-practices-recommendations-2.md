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