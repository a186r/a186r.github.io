## 智能合约中如何判断是否包含某元素

在业务中遇到需要判定某个数组中是否包含某个元素，但是solidity不提供contains方法，所以需要自己手动处理。

其实使用数组是一种非常低效的方式，并且会消耗大量资源，如果数组足够大，可能会消耗光所有的gas。
所以一般会使用mapping或者struct来做处理。

### 解决方案
有以下两种方式，可以用来判断某个元素是否存在，个人认为第二种方案更优

- #### 使用mapping处理

```
contract isContains{
    mapping(uint => bool) ids;

    function addDid(uint id) public {
        ids[id] = true;
    }

    function contains(uint id) public {
        return ids[id];
    }
}
```

- #### 使用struct处理
    由于mapping不能被枚举，也不能知道到底里面有多少个元素，所以我认为struct这种方式是更好的。
```
contract MyContract {

    struct Person {
        uint age;
        uint size;
        bool exists;
    }

    // Index of a person is its ID.
    Person[] persons;

    event PersonAdded(uint indexed id, uint age, uint size);

    function addPerson(uint _age, uint _size) public {
        Person memory person = Person(_age, _size, true);
        id = persons.push(person) - 1;

        emit PersonAdded(id, _age, _size);
    }

    function removePerson(uint _id) public {
        require(persons[_id].exists, "Person does not exist.");

        delete persons[_id];
    }
}
```

### 扩展阅读
- [Solidity CRUD- Part 1](https://medium.com/robhitchens/solidity-crud-part-1-824ffa69509a)