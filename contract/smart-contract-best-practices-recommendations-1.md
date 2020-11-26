# 智能合约开发最佳实践-1
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