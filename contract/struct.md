## 智能合约中多级struct

合约中多层struct的使用

```   
mapping(uint => Signature) singnatures;

struct Signature{
    string strsign;
    mapping(uint => Level1) level1s;
}

struct Level1{
    string a;
    string b;
    mapping(uint => Level2) strings;
}

struct Level2{
    string aa;
    string ab;
    string ba;
    string bb;
}

function setStruct() public {
    singnatures[0] = Signature({
        strsign:"strsign"    
    });
    
    Signature storage s = singnatures[0];
    
    s.level1s[0] = Level1({
        a:"a",
        b:"b"
    });
    
    Level1 storage l = s.level1s[0];
    
    l.strings[0] = Level2({
        aa:"aa",
        ab:"ab",
        ba:"ba",
        bb:"bb"
    });
    
}
```