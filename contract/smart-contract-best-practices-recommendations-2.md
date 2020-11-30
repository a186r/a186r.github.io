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

## 避免使用`tx.origin`
切勿使用`tx.origin`进行授权，另一些合约可以有一种方法来调用你的合约(例如，用户拥有一些资金)，并且
你的合约将授权你进行交易，因为你的地址在tx.origin中。

```solidity
contract MyContract {
    address owner;
    
    function MyContract() public {
        owner = msg.sender;
    }
    
    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        (bool success, ) = receiver.call.value(amount)("");
        require(success); 
    }
}

contract AttackingContract {
    MyContract myContract;
    address attacker;
    
    constructor(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }
    
    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }
}
```
你应该使用`msg.sender`来进行授权，如果另一个合约调用你的合约，则`msg.sender`将是合约的地址，而
不是调用该合约的用户的地址。

>除了授权问题之外，将来还有可能从以太坊协议中删除`tx.origin`，因此使用`tx.origin`的代码将与
>以后的版本不兼容。vitalik:不要以为`tx.origin`将继续可用或有意义。

还值得一提的是，使用`tx.origin`限制了合约之间的互操作性，因为使用`tx.origin`的合约不能被另一个
合约使用，因为合约不能是`tx.origin`

## 时间戳使用
使用时间戳执行合约中的关键功能时，主要要考虑三个方面，尤其是在涉及资金转移的操作中。

**时间戳操纵** 

请注意，矿工可以操纵该区块的时间戳，考虑下面的合约：
```solidity
uint256 constant private salt = block.timestamp;

function random(uint Max) constant private returns (uint256 result) {
    // get the best seed from randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5);
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));
    
    return uint256((h / x)) % Max + 1;
}
```
当合约使用时间戳为左随机数的种子时，矿工实际上可以在经过验证的区块后15秒钟内发布时间戳，从而有效地
使矿工预先计算初更有利于其彩票机会的期权。时间戳不是随机的，因此不应该在上下文内使用。

**15秒规则** 

以太坊黄皮书未指定在时间上可以产生多少块的约束，但确实指定了每个时间戳都应大于其父级的时间戳。以太坊
协议的客户端实现Geth和Parity都将拒绝时间戳超过15秒的块。因此评估时间戳使用情况的一个好的经验法则是：
>如果与时间相关的事件的大小可以相差15秒并保持完整性，则可以使用`block.timestamp`.

**避免使用`block.number`作为时间戳** 

可以使用`block.number`和平均区块时间来估计时间增量，但是由于区块时间可能会发生变化(例如区块重组和
难度炸弹)，因此这并不是未来的证明。在几天的销售中，15秒规则可以使你更可靠的估计时间。

## 多重继承警告
在Solidity中使用多重继承时，了解编译器如何构成继承图非常重要。
```solidity
contract Final {
    uint public a;

    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;
    
    function B(uint f) Final(f) public {

    }
    
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;
    
    function C(uint f) Final(f) public {
        
    }

    function setFee() public {
        fee =5;
    }
}

contract A is B, C {
    function A() public B(3) C(5) {
        setFee();
    }
}
```
部署合约后，编译器将从右到左线性化继承。这是合约A的线性化：
> **Final <- B <- C <- A** 

线性化的结果是fee将会是5，因为C是最衍生的合约。这看起来似乎很明显，但是可以想象一下C能够掩盖关键功能，
重新排列布尔子句并导致开发人员编写可以用合约的情况。当然，静态分析不会因过大的功能引起问题，因此必须进行
手动检查。

## 使用接口类型代替地址，以确保类型安全
当函数将合约地址作为参数时，最好传递接口或合约类型，而不是原始地址。如果在源代码中的其它位置调用该函数，
则编译器将提供附加的类型安全保证。

这里可以看到两种选择:
```solidity
contract Validator {
    function validate(uint) external returns (bool);
}

contract TypeSafeAuction {
    //good
    function validateBet(Validator _validator, uint _value) internal returns (bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    //bad
    function validateBet(address _addr, uint _value) internal returns (bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}    
```

从上面的示例中可以看出，使用上述`TypeSafeAuction`合约的好处。如果用地址参数或者`Validator`以外的
合约类型调用`validateBet()`，则编译器将会引发如下错误：

```solidity
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;
    
    function bet(uint _value) {
        // TypeError: Invalid type for argument in function call.
        bool valid = validateBet(nonValidator, _value);
    }
}
```

## 避免使用`extcodesiz`检查外部拥有的账户
通常使用以下修饰符来检查调用是从外部账户还是从合约账户进行的。
```solidity
//bad
modifier isNotContract(address _a) {
    uint size;
    assembly{
        size := extcodesize(_a)
    }
    require(size == 0);
    _;
}
```

这个想法很简单，如果一个地址包含代码，则它不是EOA，而是合约账户。但是，**合约在构造期间没有可用的源代码**,
这意味着构造函数在运行时可以调用其它合约，但是其地址的`extcodesize`返回零。下面是一个最小示例，
显示了如何规避此检查：

```solidity
contract OnlyForEOA {
    uint public flag;
    
    //bad
    modifier isNotContract(address _a) {
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }
    
    function setFlag(uint i) public isNotContract(msg.sender) {
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```

由于合约地址可以预先计算，因此如果检查在第n个块为空但大于n的某个块中部署了合约的地址，则此检查也可能
失败。

>这个问题是细微的，如果你的目标是阻止其它合约能够调用你的合约，则extcodesize检查可能就足够了。一种
>替代方法是检查(tx.origin == msg.sender)的值，尽管这样做也有缺点。
>在其它情况下，extcodesize检查可以满足你的要求。