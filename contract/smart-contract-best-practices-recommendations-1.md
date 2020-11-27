# 智能合约开发最佳实践-协议特定建议
此文档展示了编写智能合约时通常应遵循的一些模式

## 特别说明：
以下建议适用于任何以太坊上合约系统的开发

## 外部调用(External Calls)

**谨慎使用外部调用** 

调用不受信任的合约可能会带来一些意外的风险或错误。外部调用可能在该合约或它依赖的任何其它合约中执行
恶意代码。因此，每个外部调用都应视为潜在的安全风险。如果必须使用外部调用，请使用本文档其余部分中的
建议以最大程度降低风险。

**标记不受信的合约** 

与外部合约交互时，请清楚地命名与它们交互时可能不安全的变量、方法和接口，比如：

```solidity
//bad
Bank.withdraw(100);

function makeWithdrawal(uint amount) {
    Bank.withdraw(amount);
}

//good
UntrustedBank.withdraw(100);
TrustedBank.withdraw(100);

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

**避免外部调用之后的状态更改** 

无论使用原始形式调用(形式为`someAddress.call()`)或者合约调用(形式为`ExternalContract.someMethod()`),
假设可能执行恶意代码，即使ExternalContract不是恶意的，恶意代码也可以通过其调用的任何合约执行。

一种特别地危险是恶意代码可能会劫持控制流，从而由于重新进入导致漏洞。

如果你要调用不受信任的外部合约，请避免在调用后更改任何状态。这种模式有时也被称为检查-效果-互动模式
(即先检查，后改状态，最后再跟外部合约交互)。

**不要使用`transfer()`或`send()`** 

`transfer` 和 `send` 正好向接受者转发2300gas，这种硬编码的gas津贴是为了防止重入漏洞，但是这
仅仅在gas成本不变的情况下才有意义。最近，EIP1884已经在伊斯坦布尔硬分叉中实现了，EIP1884的改变之
一是增加了`SLOAD`操作的gas成本，导致合约的fallback方法的花费要高于2300gas。

建议停止使用`.transfer()`和`.send()`，使用`.call()`替代。例如：
```solidity
//bad
contract Vlunerable{
    function withdraw(uint256 amount) external {
        // 这将转发2300gas，如果接收方是合约且gas成本发生变化，这可能是不够的
        msg.sender.transfer(amount);
    }   
}

//good
contract Fixed{
    function withdraw(uint256 amount) external {
        // 这将转发所有可用的gas，务必检查返回值
        (bool success, ) = msg.sender.call.value(amount)("");
        require(success, "Transfer failed.");
    }
}
```

需要注意的是`.call()`对减轻重入攻击没有任何作用，所以必须采取其它预防措施。为了防止重入攻击，建议
使用检查-效果-互动模式。

**处理外部调用中的错误** 

Solidity提供了适用于原始地址的低级调用方法：`address.call()`,`address.callcode()`,
`address.delegatecall()`,和`address.send()`。这些低级方法从不抛出异常，但是如果调用遇到异常，
则将返回false。另一方面，合约调用(e.g.,`ExternalContract.doSomething()`)将自动传播异常（
例如，如果`doSomething`抛出异常，`ExternalContract.doSomething()`也将抛出异常）

如果选择使用低级调用方法，请确保通过检查返回值来处理调用失败的可能性。例如：
```solidity
// bad
someAddress.send(55);
// 这里有两个问题，一是它将转发所有剩余的gas，二是不检查调用结果
someAddress.call.value(55)("");
// 如果deposit()抛出异常,则原始call()将只返回false，并且交易不会被reverted（还原）
someAddress.call.value(100)(bytes4(sha3("deposit()")));

// good
(bool success, ) = someAddress.call.value(55)("");
if (!success) {
    // handle  failure code
}
ExternalContract(someAddress).deposit.value(100)();
```

**支持pull over push对于外部调用** 

外部调用可能会意外或者故意失败。为了最大程度减少此类故障导致的损失，通常最好将每个外部调用隔离到
自己的事务中，该事务可以自由地由调用的接收者发起。这与付款特别相关，在这种情况下，最好让用户自己
提取资金，而不是自动将资金推给他们（这也减少了气体限制出现问题的机会），避免在一次交易中合并多个
以太坊转账。

```solidity
//bad
contract auction {
    address highestBidder;
    uint highestBid;
    
    function bid() payable {
        require(msg.value >= highestBid);
        
        if(highestBidder != address(0)) {
            (bool success, ) = highestBidder.call.value(highestBid)("");
            require(success);
        }
        
        highestBidder = msg.sender;
        highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);
        
        if(highestBidder != address(0)) {
            // 记录用户可提取的资金
            refunds[highestBidder] += highestBid;
        }
        
        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        (bool success, ) = msg.sender.call.value(refund)("");
        require(success);
    }
}
```

**对于不受信的代码，不要使用delegatecall**

`delegatecall`用于从其他合约中调用函数，就好像它们属于调用者合约一样。因此，被调用方可以更改
调用方的状态，这可能是不安全的。下面的示例显示了使用`delegatecall`如何导致合约被破坏状态被修改。

```solidity
contract Destructor{
    function doWork() external {
        selfdestruct(0);
    }
}

contract Worker {
    function doWork(address _internalWorker) public {
        // 不安全
        _internalWorker.delegatecall(bytes4(keccak256("doWork()")));
    }
}
```

在上面的例子中，如果使用已部署的Destructor合约的地址作为参数调用`Worker.doWork()`，`Worker`
合约将会自毁，所以，应该仅将delegatecall用于被信任的合约，而不应该委托给用户提供的地址。

> 不要假设合约的余额为0。攻击者可以在创建合约之前向其地址发送以太币。合约不应该假设其初始状态包含0余额。

## 请记住可以强制将以太币发送到任何地址

攻击者可以强行将以太币发送到任何账户，并且这是无法避免的(即使使用`revert()`的fallback功能也无法阻止)。

攻击者可以创建合约，使用1wei为其提供资金，并调用`selfdestruct(victimAddress)`。`victimAddress`中
没有代码被调用，因此这是不能被阻止的，发送到矿工的地址的区块奖励也是这样的，该地址可以是任意地址。

另外，由于可以预先计算合约地址，因此可以在部署合约之前将以太币发送到某个地址。

## 请记住链上数据都是公开的

许多应用程序要求提交的数据必须保密，直到某个时间点才能使用。游戏和拍卖是主要两类。如果你要构建一个
隐私问题的应用程序，请避免要求用户过早发布信息。最好的策略是使用具有不同阶段的承诺方案：首先使用值
的哈希值进行提交，然后在后续阶段中显示值。

例如：
- 在剪刀石头布中，要求两个玩家先提交其预期动作的哈希值，然后要求两个玩家均提交其操作，如果提交的动作
与哈希值不匹配，则将其丢弃。
- 在拍卖中，要求玩家在初始阶段提交其价值的哈希值，然后在第二阶段提交其拍卖出价。
- 开发依赖于随机数生成器的应用程序时，顺序应始终为(1)玩家提交动作，(2)生成随机数，(3)玩家支付。
产生随机数的方法本身就是积极研究的领域。当前的同类最佳解决方案包括比特币区块头(通过btcrelay.org
验证)。由于以太坊是确定性协议，因此协议中的任何变量都不能作为不可预测的随机数。还要注意，矿工在某种
程度上控制着`block.blockhash()`的值。

**注意某些参与者可能离线而不返回的可能性** 

不要依赖于由特定方执行特定操作的退款或索赔程序，而没有其他方法可以取出资金。例如，在剪刀石头布游戏
中，一个常见的错误是在两个玩家都提交动作之前不进行支付。但是，恶意的玩家可以通过根本不提交自己的举
动来"困扰"对方，实际上，如果一个玩家看到了对方显示的举动并且确认自己已经输了，则根本没有理由提交自
己的举动。在状态通道解决的情况下也可能出现此问题，(1)提供规避未参与参与者的方法，可能有一定时间限
制，(2)考虑增加额外的经济激励措施，以使参与者在应有的所有情况下都可以提交信息。

**提防负数求反** 

Solidity提供了几种类型来处理带符号整数。像大多数语言一样，在Solidity中带有N位的带符号整数可以表
示`-2^(N-1)`到`2^(N-1)-1`，这意味着`MIN_INT`没有正等价物。求反的实现是找到一个数字的两个补码，
因此，对负数的取反将得出相同的数字。

对于Solidity中的所有有符号整数类型(`int8, int16, ...,int256`)都是如此。

```solidity
contract Negation{
    function negate8(int8 _i) public pure returns(int8) {
        return -_i;
    }
    
    function negate16(int16 _i) public pure returns(int16) {
        return -_i;
    }
    
    int8 public a = negate8(-128); // -128
    int16 public b =negate16(-128); // 128
    int16 public c = negate16(-32768); // -32768
}
```

处理此问题的方法是在取反之前检查`MIN_INT`的值，如果变量的值等于`MIN_INT`，则将其抛出。另一种选
择是确保使用容量更大的类型永远不会达到最大负数。(e.g. 使用`int32`代替`int16`)。

当`MIN_INT`乘以或者除以`-1`时会发生与`int`类型类似的问题。