## abi.encodeWithSelector(bytes4 selector, ...) returns (bytes memory) 方法是用来做什么的
在阅读Uniswap源码的过程中看到了这个方法，这个方法是用来做什么的呢，

- 选择器selector是函数签名的哈希值中的前4个字节。
- 函数签名由该函数的名称及参数类型所决定。
- 它允许你在不知道函数具体返回值的情况下调用函数

举例如下：

```
bytes4 private constant FUNC_SELECTOR = bytes4(keccak256("someFunc(address,uint256)"));

function func(address _contract, address _param1, uint256 _param2) view returns (uint256, uint256) {
    bytes memory data = abi.encodeWithSelector(FUNC_SELECTOR, _param1, _param2);
    (bool success, bytes memory returnData) = address(_contract).staticcall(data);
    if (success) {
        if (returnData.length == 64)
            return abi.decode(returnData, (uint256, uint256));
        if (returnData.length == 32)
            return (abi.decode(returnData, (uint256)), 0);
    }
    return (0, 0);
}
```

更常用的情况是，它允许你通过函数的字符串名称来调用函数（有点类似Java中的反射）