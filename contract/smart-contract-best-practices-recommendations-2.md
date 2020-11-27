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