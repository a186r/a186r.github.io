## Ethereum多重签名设计方案

### 比特币的多重签名方案

在探索Ethereum的多重签名设计方案之前，我们先看看比特币基于 P2SH 的多重签名方案

了解它是如何工作的，以及它如何应用与多重签名方案是很有意义的

正式的 P2SH 定义可以在 [BIP16](<https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki>) 中找到，这里有一个 [TwoOfThree.sh](<https://gist.github.com/gavinandresen/3966071>) 多重签名的示例。

步骤大致如下：

1. 根据一组公钥和一个阈值参数，生成一个多重签名地址，阈值是触发支出所需的最少签名数量
2. 为一个新的多重签名地址发送资金，将由多重签名地址产生一个 UTXO
3. 使用多重签名的 UTXO 创建一个新的原始离线交易
4. 使用私钥签署离线交易，将会返回一个16进制的字符串
5. 使用另一个私钥在这个16进制的字符串上签名，将会返回一个新的16进制字符串
6. 重复第5步，直到达到阈值，将结果发送到 UTXO 将要使用的脚本中，比特币将转移到所需的地址上

### 状态和转换

我们分析一下上面的过程，首先，选择一个 UTXO 并生成一个原始的离线交易，这是由一个私钥操作的。然后输出由另一个私钥操作，以此类推，N表示在原始交易上有N个私钥的签名，N就是多重签名钱包的签名阈值的数量。

需要注意的是这里的结果只有两种

1. 进行N个签名
2. 没有任何反应

没有中间环节就没有链上的状态转换（不包括UTXO消耗）。同样值得注意的是，上面的多重签名只能做一件事——使用UTXOs

### 拓展到Ethereum

如果目标是在Ethereum中模仿Bitcoin的多重签名方案，我们可以从上面学到很多。我们可以创建一个没有多余功能和中间状态的多重签名方案，也就是说我们的多重签名方案只有两种结果：要么执行事务，要么什么也不做。

需要注意的是，Ethereum是基于账户的，而不是像比特币一样基于UTXO，因此，它们更复杂一点。

比特币多重签名的一个重要特性是它们的实例化，一旦创建，就不能更改多重签名，这意味着合约的所有者和阈值参数将永远冻结。Ethereum将在实例化时创建不可变状态。

总而言之，我们希望Ethereum多重签名具有以下属性：

1. 二义性结果——接受交易或者立即失败
2. 功能受限——钱包可以接受交易，但是它不能做任何其他事情
3. 创建结束——一旦被创建，参数就不可更改

### 一个简单的建议

Christian Lundkvist 提出了一种符合上述特性的[多重签名方案](<https://github.com/christianlundkvist/simple-multisig/blob/master/contracts/SimpleMultiSig.sol>)：

这是他的[全文](<https://medium.com/@ChrisLundkvist/exploring-simpler-ethereum-multisig-contracts-b71020c19037>)

设置阶段是这样的：

```solidity
require(owners_.length <= 10 && threshold_ <= owners_.length && threshold_ != 0);
address lastAdd = address(0);

for (uint i=0; i<owners_.length; i++) {
  require(owners_[i] > lastAdd);
  isOwner[owners_[i]] = true;
  lastAdd = owners_[i];
}

ownersArr = owners_;
threshold = threshold_;
```

这确保owners以排序的顺序输入，并将它们发布到合约状态。这是仅有的一次ownersArr和threshold可以更改的时候



一旦创建，或者说一旦参数确定，合约只能用于一个目的：执行交易

执行过程是下面这样的：

```solidity
require(sigR.length == threshold);
require(sigR.length == sigS.length && sigR.length == sigV.length);
// Follows ERC191 signature scheme
bytes32 txHash = keccak256(byte(0x19), byte(0), this, destination, value, data, nonce);
address lastAdd = address(0); // cannot have address(0) as an owner   

for (uint i = 0; i < threshold; i++) {
  address recovered = ecrecover(txHash, sigV[i], sigR[i], sigS[i]);
  require(recovered > lastAdd && isOwner[recovered]);
  lastAdd = recovered;
}
// If we make it here all signatures are accounted for
nonce = nonce + 1;
require(destination.call.value(value)(data));
```

这将检查是否提供了threshold签名，并根据交易参数生成hash

然后，他检查每一个签名，并验证它是由计算值大于前一个所有者的所有者创建的，这里的比较有点随意，这是一个简单的十六进制字符的比较。这个约束进一步限制了攻击面，但是可能没有必要。

如果所有检查都通过，则使用提供的参数在合约中进行交易的调用，增加nonce，以防止重放攻击，需要注意的是，只有增加nonce这里是唯一的一次状态改变。理论上，这个合约的攻击面被限制为这个nonce值递增，这个攻击面非常小。

### Ethereum的优势

这个方案正是Grid+团队正在寻找的。在43行代码中，它就像多重签名合约一样简单。它只有一个函数，即在Ethereum上创建交易。因为任何复杂的函数调用都可以执行，所以**这个多重签名函数就像一个标准的钱包一样，但是需要多个签名**。

以下是代码：

```solidity
pragma solidity 0.4.18;
contract SimpleMultiSig {

  uint public nonce;                // (only) mutable state
  uint public threshold;            // immutable state
  mapping (address => bool) isOwner; // immutable state
  address[] public ownersArr;        // immutable state

  function SimpleMultiSig(uint threshold_, address[] owners_) public {
    require(owners_.length <= 10 && threshold_ <= owners_.length && threshold_ != 0);

    address lastAdd = address(0);
    for (uint i=0; i<owners_.length; i++) {
      require(owners_[i] > lastAdd);
      isOwner[owners_[i]] = true;
      lastAdd = owners_[i];
    }
    ownersArr = owners_;
    threshold = threshold_;
  }

  // Note that address recovered from signatures must be strictly increasing
  function execute(uint8[] sigV, bytes32[] sigR, bytes32[] sigS, address destination, uint value, bytes data) public {
    require(sigR.length == threshold);
    require(sigR.length == sigS.length && sigR.length == sigV.length);

    // Follows ERC191 signature scheme: https://github.com/ethereum/EIPs/issues/191
    bytes32 txHash = keccak256(byte(0x19), byte(0), address(this), destination, value, data, nonce);

    address lastAdd = address(0); // cannot have address(0) as an owner
    for (uint i = 0; i < threshold; i++) {
        address recovered = ecrecover(txHash, sigV[i], sigR[i], sigS[i]);
        require(recovered > lastAdd && isOwner[recovered]);
        lastAdd = recovered;
    }

    // If we make it here all signatures are accounted for
    nonce = nonce + 1;
    require(destination.call.value(value)(data));
  }

  function () public payable {}
}
```

多重签名钱包应该成为存储大量加密货币的实际标准，它们应该尽可能的简单、安全、并对用户友好，这样它们才能成为常用的工具，即使是那些想要确保资金不受单点故障影响的个人也可以很方便的使用它们。