## 智能合约中`external`和`public`的最佳实践

除了`public`之外，以太坊还引入了`external`修饰符，两者都可以在合约的内部和外部进行调用，后者
通过`this.f()`模式调用。根据文档：
> `external`方法在接收大量数组是表现会更好。

但是使用`external`和`public`的最佳实践是什么呢?

先用一个简单的例子解释一下：
```solidity
pragma solidity ^0.4.12;

contract Test {
    function test(uint[20] a) public pure returns (uint) {
        return a[10]*2;
    }
    
    function test2(uint[20] a) external pure returns (uint) {
        return a[10]*2;
    }
}
```
调用两个函数时，我们会发现，`public`函数会花费`552`gas，而`external`则只花费了`317`gas,
造成这种区别的原因是，在公共函数中，`Solidity`立即将数组参数复制到内存中，而`external`函数可以
直接从`calldata`中读取。内存分配很昂贵，而从`calldata`读取很便宜。

`public`函数需要将所有参数写入内存的原因是`public`函数可以在内部调用，实际上这是与外部 调用完全
不同的过程。内部调用是通过代码中的跳转执行的，而数组参数是通过指向内存的指针在内部传递的。因此，当
编译器为内部函数生成代码时，该函数期望变量存在与内存中。

对于`external`函数，编译器不需要允许内部调用，因此它允许直接从calldata中读取参数，从而节省了复制的
步骤与成本。

对于最佳实践，如果你希望仅在外部调用该函数，则应使用`external`，如果需要在内部使用该函数，则应使用
`public`。使用`this.f()`模式几乎是没有任何意义的，因为这需要执行真正的`CALL`，这很昂贵，同样，通过
此方法传递数组要昂贵得多。

从本质上讲，只要你仅在外部调用函数，并传入大型数组，就可以看到`external`的性能优势。

但是需要注意的一点是，`external`修饰的函数，入参数量的限制可能会更少(少于16个)，更容易引发
`CompilerError: Stack too deep, try removing local variables`

下面是函数修饰符的具体区分：
- `public` - 所有人都可以访问
- `external` - 不能被内部访问，只能外部访问
- `internal` - 只能被此合约和由此合约衍生的合约访问
- `private` - 只能从此合约访问
