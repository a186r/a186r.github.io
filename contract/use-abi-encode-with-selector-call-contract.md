## 使用abi.encodeWithSelector调用其他合约的函数
`abi.encodeWithSignature`和`abi.encodeWithSelector`主要使用场景都是函数调用，下面是一个具体的实践

先写一个简单的合约MyContract
```
pragma solidity >=0.4.21 <=0.7.0;

contract MyContract{
    
    function function1() public {}
    
    function getBalance(address account) public view returns (uint) {}
    
    function getValue(uint value) public pure returns (uint) {
        return value;
    }
    
}
```

使用Contract合约调用MyContract合约中的getValue方法

```
pragma solidity >=0.4.21 <=0.7.0;

import "./MyContract.sol";

contract Contract{
    
    MyContract myContract = new MyContract();
    
    function getSelector() public view returns (bytes4, bytes4) {
        return (myContract.function1.selector, myContract.getBalance.selector);
    }
    
    function callGetValue(uint x) public view returns (uint) {
        bytes4 selector = myContract.getValue.selector;
        
        bytes memory data = abi.encodeWithSelector(selector, x);
        
        (bool success, bytes memory returnedData) = address(myContract).staticcall(data);
        
        require(success, "FAILED");
        
        return abi.decode(returnedData, (uint256));
    }
}
```