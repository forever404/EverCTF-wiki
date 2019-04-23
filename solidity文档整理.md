<h2 align=center>关于libc与rop的思考</h2>


### 文档整理  
---
#### 1. 合约的结构  

- 状态变量(State Variable)  
  

&emsp;&emsp;状态变量指的是那些直接声明在函数外的变量，他们被永久的储存在合约里。  

```javascript
pragma solidity >= 0.4.0 < 0.6.0;

contract SimpleStorage{
    uint storedData;	//State variable
}
```

 - 函数(Function)  

&emsp;&emsp;solidity里的函数与Javascript极为相似，但是包含更多的修饰词和可见性限制,也可以有多个返回值。  

- 函数修改器(Function Modifier)  

&emsp;&emsp;函数修改器主要是以声明的形式来修改函数的语义，比如给函数的出发增加限制条件或者验证。  

```javascript
pragma solidity >= 0.4.0 < 0.6.0;
contract Purchase{
    address public seller;
    
    modifier onlySeller() {
        require(msg.sender == seller,"only seller can call this");
        -;
    }
    
    function abort() public view onlySeller{	// Modidier usage
        // ...
    }
}
```

- 事件(Event)  

&emsp;&emsp;事件是**EVM logging**的便利接口。当事件被触发时，可将部分数据记录到区块链上。  

```javascript
pragma solidity >= 0.4.0 < 0.6.0;

contract SimpleAuction{
    event HighestBidIncreased(address bidder, uint amount); //Event
    
    function bid() public payable{
        // ...
        emit HighestBidIncreased(msg.sender,msg.value); //Triggering event
    }
}
```

- 结构(Struct)  

&emsp;&emsp;结构体与c语言极为相似  

```c
pragma solidity >= 0.4.0 < 0.6.0;

contract Ballot{
    struct Voter{ // Struct
        uint weight;
        bool voted;
        address delegate;
        uint vote;
    }
}
```

- 枚举(Enum)  

&emsp;&emsp;枚举与C++中的枚举类似，都是自定义类型，你可以认为这是一个常量集合。  

```c++
pragma solidity >= 0.4.0 < 0.6.0;

contract Purchase {
    enum State {
        Created, Locked, Inactive		//Enum
    }
}
```

---

#### 2. 数据类型  

- 布尔(Booleans)  

&emsp;&emsp;与通常的语言一样，**bool** 包含 ***true*** 和 ***false*** 两种常量。  

**Operators**:  

1.	**!**  非  
2.	**&&** 与  
3.	**||**  或  
4.	**==**  等于  
5.	**!=**  不等于  

> **||** 和 **&&** 遵守短路定律，这意味着表达式  

```javascript
f(x) || g(y)
```
&emsp;&emsp;如果f(x)为真，g(y)将不参与运算，尽管这可能有副作用。  

- 整数(Integers)  

&emsp;&emsp;**int** / **uint** 分别是有符号和无符号整数，他们具有可变的内存体积。关键字uint8到uint256与int8到uint8相对应。数字后缀代表的是变量的内存大小，uint8指的是**8bits**的无符号整数。并且uint与int是uint256与int256的别名(alias)。  

**Operator**:  

1. 比较： **<= , < , == , != , >= , >** (表达式的值为bool)
2. 位运算： & , | , ^ , ~  
3. 移位运算： << , >>  
4. 算数运算： +  ,  -   ,  *  ,  /  

> 整数的大小范围在solidity中十分严格，例如uint32代表0到2**32 - 1之间的数，如果结果超出这个范围，那么可能造成上溢或者下溢，这可能会给合约造成严重的安全隐患。  



- 地址(Address)  

&emsp;&emsp;地址类型是较为特殊的变量类型，这中变量对应一个合约或者账户(本质上合约就是一个账户)，他主要包含两种风格：  

> address: 包含20byte的值  （以太坊地址）
> address payable: 与address一样，但是包含transfer和send两个成员  

&emsp;&emsp;两者的主要区别是，后者可以就收以太币(Ether)，但是address却不能，这里一定要注意，尤其是在写攻击合约的时候。  

&emsp;&emsp;address payable 到 address的隐式转换是允许的，但是反过来却不行，地址字面量能够被隐式的转换为address payable  

&emsp;&emsp;int 整数字面量(integer literals) bytes20以及合约 类型都可以被被显式的转换为address类型。  

> int 字面量 和bytes20 想要转换为address payable必须满足地址本身代表的合约或者账户的fallback(回滚)函数必须是payable的。当然，如果address变量的fallback函数是payable的，那么显式的转换也是可以的。

> warning:
>
> > 如果你想要将一个大的bytes类型转换为地址，比如bytes32,这个时候address会截尾，为了避免二义性，你必须显式的自行进行截断的选择。

```javascript
b = 0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC;

address(uint160(bytes20(b))) 		//这时结果为0x111122223333444455556666777788889999aAaa

address(uint169(uint256(b)))		//这时结果为0x777788889999AaAAbBbbCcccddDdeeeEfFFfCcCc

```

- 地址的成员(Members of Address)  

&emsp;&emsp;上面提到balance和transfer是address payable类型的两个成员。  

&emsp;&emsp;如果地址是payable的，那么我们可以查询地址剩余的Ether或者向他打钱，例如：  

```javascript
address payable x = address(0x123);
address myAddress = address(this);

if(x.balance < 10 && myAddress.balance >= 10){
	x.transfer(10);    
}
```
&emsp;&emsp;这个时候我们就向x地址打了10块钱，balance实际上是调用了成员对象本身的getter()函数，getter能够返回一个状态变量的值。  

> transfer函数将会在ether不足或者对方拒绝收钱时执行失败，这个时候transfer()将会回滚，注意这里很重要，有转账功能的还有send()函数和<address>.call.value()()函数，后两者在失败时转出的钱不会回滚，这就有可能导致Reentrancy漏洞。  

| 函数 | 处理 | 消耗 |
|:---|:---|:---|
|<address>.transfer()|失败时抛出异常，并且回滚状态|消耗2300 Gas (not adjustable) 可防止回滚|
|<address>.send()|失败时返回错误|消耗2300 Gas (not adjustable) 不防止回滚|
|<address>.call.value().gas()()|失败时返回错误|消耗Gas可调节 不防止回滚|

> <address>.send()与<address>.call.value()()函数在转账过程中发生异常时，不能有效回滚，导致Reentrancy的攻击无法防御，因此我们在交易时应当使用transfer()函数。

> 关于call()  delegatecall()   staticcall()
> 为了不依赖于ABI来调用合约的接口，或者更为直接的调用其他合约的方法，solidity提供了call，delegatecall，staticcall。他们都接受一个**bytes memory**类型的参数，并且返回bool类型和被调用方法的返回值。方法abi.encode,abi.encodePacked,abi.encodeWithSelector以及abi.encodeWithSignature 可以被用来将数据编码结构化。

```javascript
bytes memory payload = abi.encodeWithSignature("register(string)","MyName");

(bool success, bytes memory returnData) = address(nameReg).call(payload);

require(success);
```

#### 关于call()与delegatecall()的区别：  

&emsp;&emsp;这里要强调一下call与delegatecall可能导致的问题，两者都是底层调用，但是两者的上下文不同，call所代表的上下文是被调用合约实例本身，而delegatecall则是该方法调用的发起者。

![delegatecall](https://github.com/Explainaur/hexo-blog/blob/master/source/pictures/eth_writeUp0.png?raw=true)  

&emsp;&emsp;因此这时候我们应当注意状态的转换，尽量少的使用底层调用。  

> 注：
> 作为底层调用，如果你调用了任何未知的恶意合约，相当于你将控制权交给了他，这有可能导致该恶意合约回调你的合约，所以你的合约状态变量可能会被恶意修改。通常情况下我们应当创建一个合约实例,如: x.f()

&emsp;&emsp;call()的用法实例：  

1. 通过gas()函数修改器来调整Gas  

```javascript
address(nameReg).call.gas(1000000)(abi.encodeWithSignature("register(string)", "MyName"));
```

2. 通过value()函数修改器来调整Ether：  

```javascript
address(nameReg).call.value(1 ether)(abi.encodeWithSignature)("register(string)","MyName");
```

3. 这两种函数修改器可以结合：  

```javascript
address(nameReg).call.gas(1000000).value(1 ether)(abi.encodeWithSignature("register(string)", "MyName"));
```

&emsp;&emsp;类似地，函数delegatecall也可以被如此调用，区别是：此函数只使用给定地址的代码，其他方面比如(storage,balance...)都是当前调用合约，这一点上面我们说过了。使用delegatecall的目的主要是使用其他合约的library中的方法。但是用户需要确定两合约的内存布局适合是的。  

&emsp;&emsp;对于staticcall，他与call十分相似，但这个方法会revert(恢复调用前状态)如果被调用方法修改了状态变量。

> delegatecall()不支持value()修改器  


>所有的合约都能被显式转换为地址，因此可以使用address(this).balance来查看当前合约的存款。  

---

- 合约类型(Contract)  

&emsp;&emsp;与C++类似，每种合约都是一种数据类型，子类合约可以隐式的转换为父类或超类合约，并且能够显式的转换为address类型。  

&emsp;&emsp;与address类似，只有fallback函数是payable的合约才能转化为address payable。转换的方式依旧是**address(contract)** 而不是 **address payable(contract)**.  

&emsp;&emsp;你可以声明一个合约类型的局部变量，那么你就可以调用那个合约的方法，如：  

```javascript
contract A {
	uint x;
    contrusctor(uint a) internal{
        x = a;
    }
}

contract B {
    A a = A(2);			//调用了A的构造函数
    function echoA() public view returns(uint){
        return a.x();		//返回a.x的值
    }
}
```

&emsp;&emsp;但是假如你想要调用已经存在的一个实例，比如想要攻击已经在链上的一个合约，这个时候你可以：  

```javascript
contract Fuck {
	address target_contract_addr = "0x123";		//首先获得攻击目标的地址
    
    TargetContract x = TargetContract(target_contract_addr);		//将地址注入构造函数
    
    x.balance();		//进行你想做的攻击
    
    //...
}
```

> 你可以用type(c)来获得c合约的类型。  

---

- 定长数组(Fixed-size byte arrays)  

&emsp;&emsp;bytes1到bytes32能储存一列数，数组长度是从1到32，byte是byte1的alias。  

Operator：  

1. 比较 <= , < , == , != , >= , > 结果返回bool  
2. 位运算： & , | , ^ , ~   
3. 移位运算： << , >>  
4. 寻址运算： x[k] 获得数组x的第i+1个数据  

> 成员对象.length是只读的，不可修改  

- 变长数组(Dynamically-sized byte array)  

&emsp;&emsp;bytes是动态大小的数组，不是值类型。  
&emsp;&emsp;string是UTF-8-encode 的string类型，不是值类型。  

---

- 字面量(Literals)  

&emsp;&emsp;字面量包括地址字面量(Address Literals)，字符串字面量(String Literals)和有理字面量(Rational Literals)以及整数字面量(Interege Literals),通俗来讲字面量就是常数，而且其精度无限(与其本身长度有关)，但是当字面两转化为非字面量时，精度可能会损失。  

> 5/2对于字面量来说是2.5，而对于uint来说是2。字面量参与非字面量进行运算时，其类型必须相同，如:  

```javascript

uint128 a = 1;
uint128 b = 2.5 + a;		//这样写是会报错的
```

---

- 函数(Function)  

&emsp;&emsp;函数的用法与js极为相似，只是有些可见性关键字需要解释一下，首先来看一下声明格式：  

```javascript
function (<parameter types>) {internal|external} [pure|view|payable] [<returns types>]
```

&emsp;&emsp;函数中假如有返回值，则返回值的类型不能省略，如果返回值缺省，那么整个**returns()**的部分都应该省略。  

&emsp;&emsp;默认情况下函数的可见性是internal，但是我自己在尝试时发现，如果可见性关键字缺省则会导致报错。  

&emsp;&emsp;下面我们来解释一下可见性与访问控制的问题：  

&emsp;&emsp;函数的可见性分为四种：**public private internal external** .

#### internal 

&emsp;&emsp;internal调用，实现时转为简单的EVM跳转，所以他能够直接访问上下文的数据，对于引用传递是十分高效，例如memory之间的值传递，实际上是引用的传递(妈耶，storage和memory又是坑，不同版本真是令人窒息)。  

&emsp;&emsp;当前代码单元内，比如同一个合约内的函数，引入的library库，以及父类函数的直接调用即为internal调用，比如：  

```javascript
pragma solidity >=0.4.0 < 0.6.0;

contract test{
    function a() internal {}

    function b() internal {
        a();
    }
}
```

&emsp;&emsp;在上述代码中的b()对a()的调用即为internal方式调用，函数在不显式声明访问类型时,以目前的版本来看会报错。

#### external  

&emsp;&emsp;external调用实现了合约的外部消息调用。所以合约在初始化时不能以external的方式调用自身函数，因为此时合约仍未构造完成，此处可类比struct类型，一个结构体不能包含自身对象。但是可以以this的方式强制进行external调用。  

```javascript
pragma solidity >= 0.4.0 < 0.6.0;
contract test{
    function  a() external {}

    function b() public {
        a();  //此时会报错
    }

    contract ext{
        function callA(test tmp) public {
            tmp.a();
        }
    }
}
```
#### public  

&emsp;&emsp;public的特点是，函数既可以以internal方式调用，也可以用internal方式调用。public函数可以被外部接口访问，是合约对外接口的一部分。 
```javascript
pragma solidity >= 0.4.0 < 0.6.0

contract test{
    function fun1() public{}

    funciton fun2() public {
        fun1();
        this.fun2();
    }
}
```
&emsp;&emsp;可以看到没有报错，既然public这么舒服，那为啥我还要用external？？？  

&emsp;&emsp;经过对比后我们可以发现，external方法消耗的gas要比public少，因为Solidity在调用public函数时会将代码复制到EVM的内存中，而external则是以calldata的方式进行调用的。内存分配在EVM中是十分宝贵的，而读取calldata则十分廉价，因此在处理大量外部数据，并反复调用函数时，应当考虑用external方法。  

&emsp;&emsp;这里应当注意的是，public属于可见性。函数的可见性分为四种：**public private internal external** .  

#### private  

&emsp;&emsp;对于private，与internal的区别是，private的方法在子类中无法调用，即使被声明为private也不能阻止数据的查看。访问权限仅仅是限制其他合约对函数的访问和数据修改的权限。而private方法也默认以internal的方式调用。  

```javascript
pragma solidity >= 0.4.0 < 0.6.0;

contract test{
    function fun1() private{}

    function fun2() public{
        fun1();
        //this.fun1()
    }
}

//合约的继承为is，这一点很容易理解，如果你明白设计模式的话，实际上继承是A is B 的关系,我很喜欢这种写法。

contract ext is test{   
    function callFun() public {
        //fun1();   
        fun2();
    }
}
```
&emsp;&emsp;这里我们可以明确的看到private的效果，和internal类似，但是代价会更大。  

&emsp;&emsp;然而 **public** 与 **private** 还可以被作用于其他的变量，用于设置外部访问权限。  

&emsp;&emsp;请大家务必不要弄混 **调用方式** 与 **可见性(visable)** 。  

> 关于 view pure constant  

&emsp;&emsp;在0.4.1之前只有constant这一种可爱的语法，就是有一些屁事很多的人觉得constant指的是变量，作用于函数不太合适，所以就把constant拆成了view和pure。  

&emsp;&emsp;在Solidity中，**constant view pure** 的作用是告诉编译器，函数 **不改变**，**不读取**状态变量，这样一来函数的执行就不再消耗gas了，因为不再需要矿工去验证。  

&emsp;&emsp;然而这三个东西有点有意思，在官方文档中用 **restrictive** 这一词来对函数的严格性进行描述，在函数类型转换时对严格行有一定的要求，高严格性函数可以被转化为低严格性函数：  

- pure 类型可被转化为 **view** 和 **non-payable** 函数  

- view 类型可被转化为 non-payable 函数  

- payable 类型可被转化为 non-payable 函数  


Member:  

1. selector  返回ABI函数选择器。  
2. gas(uint) 返回一个函数对象，当被调用时将会发送具体数目的Gas给目标函数。  
3. value(uint)返回一个函数对象了，当被调用时将会发送具体的wei给目标函数，或者使用value(1 ether)的方式来发送以太币。  

&emsp;&emsp;我们来看一下用法示例：  

```javascript
pragma solidity >=0.4.16 <0.6.0;

contract Example {
  function f() public payable returns (bytes4) {
    return this.f.selector;
  }
  function g() public {
    this.f.gas(10).value(800)();
  }
}
```

&emsp;&emsp;下面是internal关键字的用法示例：  

```javascript
pragma solidity >=0.4.16 <0.6.0;

library ArrayUtils {
  // internal functions can be used in internal library functions because
  // they will be part of the same code context
  function map(uint[] memory self, function (uint) pure returns (uint) f)
    internal
    pure
    returns (uint[] memory r)
  {
    r = new uint[](self.length);
    for (uint i = 0; i < self.length; i++) {
      r[i] = f(self[i]);
    }
  }
  function reduce(
    uint[] memory self,
    function (uint, uint) pure returns (uint) f
  )
    internal
    pure
    returns (uint r)
  {
    r = self[0];
    for (uint i = 1; i < self.length; i++) {
      r = f(r, self[i]);
    }
  }
  function range(uint length) internal pure returns (uint[] memory r) {
    r = new uint[](length);
    for (uint i = 0; i < r.length; i++) {
      r[i] = i;
    }
  }
}

contract Pyramid {
  using ArrayUtils for *;
  function pyramid(uint l) public pure returns (uint) {
    return ArrayUtils.range(l).map(square).reduce(sum);
  }
  function square(uint x) internal pure returns (uint) {
    return x * x;
  }
  function sum(uint x, uint y) internal pure returns (uint) {
    return x + y;
  }
}
```

&emsp;&emsp;接下来是external关键字的用法：  

```javascript
pragma solidity >=0.4.22 <0.6.0;

contract Oracle {
  struct Request {
    bytes data;
    function(uint) external callback;
  }
  Request[] requests;
  event NewRequest(uint);
  function query(bytes memory data, function(uint) external callback) public {
    requests.push(Request(data, callback));
    emit NewRequest(requests.length - 1);
  }
  function reply(uint requestID, uint response) public {
    // Here goes the check that the reply comes from a trusted source
    requests[requestID].callback(response);
  }
}

contract OracleUser {
  Oracle constant oracle = Oracle(0x1234567); // known contract
  uint exchangeRate;
  function buySomething() public {
    oracle.query("USD", this.oracleResponse);
  }
  function oracleResponse(uint response) public {
    require(
        msg.sender == address(oracle),
        "Only oracle can call this."
    );
    exchangeRate = response;
  }
}
```

> Lambda(匿名)和inline(内联)函数暂时不支持，在后续版本即将推出。  

- 引用类型(Reference Type)  

&emsp;&emsp;引用与C++中的引用类似，即一个量可以通过多个别名修改，与值类型相比较，后者可以直接得到一个拷贝对象。因此，在使用引用类型时应当格外小心。目前，引用类型包括结构，数组和映射。如果你使用一个引用类型，你必须显式的声明他的储存位置：  

> memory 生命周期为一个函数调用，只在EVM内存中存在  
> storage 生命周期无限，与状态变量一起储存在区块链上  
> calldata 包含函数参数的特殊数据位置，仅可用于external函数调用参数  

&emsp;&emsp;更改Data Location的赋值或类型转换将会引发自动复制操作，若两者的Data Location类型相同，那么只在两者均为storage的某些情况下才会引发复制临时对象。  

#### 数据位置(Data Location)  

&emsp;&emsp;如上面提到，引用类型必须显式的添加"data location"的声明，即memory storage calldata.  
&emsp;&emsp;Calldata只对external函数的参数有效，并且对于此类型的参数是必须的。Calldata类型的变量的存储位置是不可修改的，非持久的储存函数参数的区域，行为与memory十分类似。  

#### 内存区域和分配行为  

&emsp;&emsp;数据位置不仅与数据的持久性有关，还与赋值的语义有关：  

1. 当赋值行为在memory(或者calldata)与storage之间时，会直接创造一个拷贝对象  
2. 当赋值行为是memory与memory时，仅仅创造一个引用，这意味着如果修改其中一个memory变量，将会导致所指向的同一位置的内存数据的修改(与C语言的指针类似)  
3. 当storage赋值给local storage(函数中的storage)之间时，此时也仅仅分配一个引用  
4. 其他所有赋值给storage或状态变量的操作都会创造一个拷贝对象，即使给storage的局部变量仅仅是个引用。下面的例子展现了这几种特性：  

```javascript
pragma solidity >=0.4.0 <0.6.0;

contract C {
    uint[] x; // the data location of x is storage

    // the data location of memoryArray is memory
    function f(uint[] memory memoryArray) public {
        x = memoryArray; // works, copies the whole array to storage
        uint[] storage y = x; // works, assigns a pointer, data location of y is storage
        y[7]; // fine, returns the 8th element
        y.length = 2; // fine, modifies x through y
        delete x; // fine, clears the array, also modifies y
        // The following does not work; it would need to create a new temporary /
        // unnamed array in storage, but storage is "statically" allocated:
        // y = memoryArray;
        // This does not work either, since it would "reset" the pointer, but there
        // is no sensible location it could point to.
        // delete y;
        g(x); // calls g, handing over a reference to x
        h(x); // calls h and creates an independent, temporary copy in memory
    }

    function g(uint[] storage) internal pure {}
    function h(uint[] memory) public pure {}
}
```

---

- 数组(Array)  

&emsp;&emsp;数组的用法上面介绍的都差不多了，这里需要注意的是solidity中的数组的声明方式与通常的语言不同，他的第一个下标是一个数组的位置，第二个下标是数组中元素的位置：  

```javascript
uint[][5] x memory;		//一个由5个储存uint类型的动态数组被写入数组x		这种方式与其他语言相反

X[2][1];		//代表第三个数组的第二个元素

T[5] a;			//T本身可以是个数组，那么a[2]就代表T类型的变量
```

&emsp;&emsp;数组元素可以是任意类型，包括映射(mapping)与结构(struct)类型。但是通常情况下这种使用有限制，因为mapping与struct必须存储在storage数据区域内。  

&emsp;&emsp;将状态变量数组声明为public是可行的，并且solidity将会为其创建一个getter接口。那么数组的数字索引将会是getter()的参数。  

&emsp;&emsp;当数组寻址且超过其长度范围时，将会导致一个失败断言(failing assertion).你可以使用.push()与.pop()函数来向末尾增加或弹出元素(与C++的STL类似)，也可以直接修改.length成员来修改数组的长度。  

> 关于bytes与string  

&emsp;&emsp;bytes与string是特殊测数组，bytes与byte[]类似，但是他在calldata与memory存储区域内将会被打包的更紧致。string与bytes的区别是，string不能访问.length也不能进行寻址操作。  

&emsp;&emsp;solidity没有字符串操作函数，但有第三方字符串库。还可以使用keccak256-hash函数来比较两个字符串：  

```javascript
keccak256(abi.encodePacked(s1)) == keccak256(abi.encodePacked(s2))
```

&emsp;&emsp;并通过`abi.encodePacked(s1, s2)`来连接两个字符串。  

&emsp;&emsp;你应当尽量的使用bytes而不是bytes[]，因为bytes[]在两个元素之间增加了31个填充字节。作为一般规则，对任意长度的原始byte数据使用bytes，对任意长度字符串（UTF-8）数据使用string。  

> 加入你一定要对string对象进行.length或者寻址操作，那么你应当先把他强制转换为bytes类型，如：  

```javascript
string s;

uint len = bytes(s).length;

bytes(s)[7] = 'x';
```

#### 为数组分配内存  

&emsp;&emsp;为数组分配内存与C++类似，要使用new关键字在内存中创建运行时确定长度的数组,如：  

```javascript
pragma solidity >=0.4.16 <0.6.0;

contract C {
    function f(uint len) public pure {
        uint[] memory a = new uint[](7);
        bytes memory b = new bytes(len);
        assert(a.length == 7);
        assert(b.length == len);
        a[6] = 8;
    }
}
```

#### 数组成员  

**Length**:   
	数组的length成员包含了数组元素的个数，这个长度在内存中一旦确定是不可变的(不包括动态数组)，对于动态数组，给length重新赋值能够修改其长度。当寻址超出长度之外时，你将会引发一个失败断言。新增加的长度的值被初始化为0,你可以通过delete关键字删除单个的元素来减少数组的长度。如果你尝试改变一个非动态数组的length，你会得到一个`Value must be an lvalue`错误。  

**push**:  
&emsp;&emsp;动态数组以及bytes与string拥有push()成员函数，你可以使用push来向数组末尾添加一个元素，若参数为空则默认为0,该函数返回新的数组长度。  

**pop**:  
&emsp;&emsp;动态数组以及bytes与string拥有pop()成员函数，你可以使用pop来删除数组末尾的最后一个元素。  

> 注意：  
>
> > 这里一定要注意动态数组的下溢问题(underflow)，假如你对一个空数组进行`<d-array>.length--`操作，那么这将会导致数组的长度变为2**256 - 1,这意味着你将可以访问内存中的任意变量，也可能导致某些逻辑判断的步骤出错。  

&emsp;&emsp;增加数组的长度将会消耗固定的Gas，因为新增的元素被初始化为0,当减少长度时则消耗线性的Gas(但通常情况下要比线性糟糕)，	因为包含了显式的删除与清理元素的步骤，即调用delete关键字。  

> 目前还不能在external函数中使用数组的数组，但是在public函数中是支持的。  

> 在拜占庭(Byzantium)之前的EVM版本中，无法访问函数调用返回的动态数组。如果调用返回动态数组的函数，请确保使用设置为拜占庭模式的EVM。关于拜占庭请参考白皮书中的拜占庭将军问题，很有意思。

&emsp;&emsp;数组用法实例如下：  

```javascript
pragma solidity >=0.4.16 <0.6.0;

contract ArrayContract {
    uint[2**20] m_aLotOfIntegers;
    // Note that the following is not a pair of dynamic arrays but a
    // dynamic array of pairs (i.e. of fixed size arrays of length two).
    // Because of that, T[] is always a dynamic array of T, even if T
    // itself is an array.
    // Data location for all state variables is storage.
    bool[2][] m_pairsOfFlags;

    // newPairs is stored in memory - the only possibility
    // for public contract function arguments
    function setAllFlagPairs(bool[2][] memory newPairs) public {
        // assignment to a storage array performs a copy of ``newPairs`` and
        // replaces the complete array ``m_pairsOfFlags``.
        m_pairsOfFlags = newPairs;
    }

    struct StructType {
        uint[] contents;
        uint moreInfo;
    }
    StructType s;

    function f(uint[] memory c) public {
        // stores a reference to ``s`` in ``g``
        StructType storage g = s;
        // also changes ``s.moreInfo``.
        g.moreInfo = 2;
        // assigns a copy because ``g.contents``
        // is not a local variable, but a member of
        // a local variable.
        g.contents = c;
    }

    function setFlagPair(uint index, bool flagA, bool flagB) public {
        // access to a non-existing index will throw an exception
        m_pairsOfFlags[index][0] = flagA;
        m_pairsOfFlags[index][1] = flagB;
    }

    function changeFlagArraySize(uint newSize) public {
        // if the new size is smaller, removed array elements will be cleared
        m_pairsOfFlags.length = newSize;
    }

    function clear() public {
        // these clear the arrays completely
        delete m_pairsOfFlags;
        delete m_aLotOfIntegers;
        // identical effect here
        m_pairsOfFlags.length = 0;
    }

    bytes m_byteData;

    function byteArrays(bytes memory data) public {
        // byte arrays ("bytes") are different as they are stored without padding,
        // but can be treated identical to "uint8[]"
        m_byteData = data;
        m_byteData.length += 7;
        m_byteData[3] = 0x08;
        delete m_byteData[2];
    }

    function addFlag(bool[2] memory flag) public returns (uint) {
        return m_pairsOfFlags.push(flag);
    }

    function createMemoryArray(uint size) public pure returns (bytes memory) {
        // Dynamic memory arrays are created using `new`:
        uint[2][] memory arrayOfPairs = new uint[2][](size);

        // Inline arrays are always statically-sized and if you only
        // use literals, you have to provide at least one type.
        arrayOfPairs[0] = [uint(1), 2];

        // Create a dynamic byte array:
        bytes memory b = new bytes(200);
        for (uint i = 0; i < b.length; i++)
            b[i] = byte(uint8(i));
        return b;
    }
}
```

---


- 结构(struct)  

&emsp;&emsp;Solidity提供了一种声明新的类型的方法，即struct。struct与C/C++一样，用法如下：  

```javascript
pragma solidity >=0.4.11 <0.6.0;

contract CrowdFunding {
    // Defines a new type with two fields.
    struct Funder {
        address addr;
        uint amount;
    }

    struct Campaign {
        address payable beneficiary;
        uint fundingGoal;
        uint numFunders;
        uint amount;
        mapping (uint => Funder) funders;
    }

    uint numCampaigns;
    mapping (uint => Campaign) campaigns;

    function newCampaign(address payable beneficiary, uint goal) public returns (uint campaignID) {
        campaignID = numCampaigns++; // campaignID is return variable
        // Creates new struct in memory and copies it to storage.
        // We leave out the mapping type, because it is not valid in memory.
        // If structs are copied (even from storage to storage), mapping types
        // are always omitted, because they cannot be enumerated.
        campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0);
    }

    function contribute(uint campaignID) public payable {
        Campaign storage c = campaigns[campaignID];
        // Creates a new temporary memory struct, initialised with the given values
        // and copies it over to storage.
        // Note that you can also use Funder(msg.sender, msg.value) to initialise.
        c.funders[c.numFunders++] = Funder({addr: msg.sender, amount: msg.value});
        c.amount += msg.value;
    }

    function checkGoalReached(uint campaignID) public returns (bool reached) {
        Campaign storage c = campaigns[campaignID];
        if (c.amount < c.fundingGoal)
            return false;
        uint amount = c.amount;
        c.amount = 0;
        c.beneficiary.transfer(amount);
        return true;
    }
}
```

&emsp;&emsp;该合同不提供众筹合同的全部功能，但它包含理解结构所必需的基本概念。结构可被用于映射(mapping)或者数组(array)类型，并且结构也可以包含映射与数组。  

&emsp;&emsp;与C++类似，结构不可包含其自身成员对象，这个限制是必须的，否则无线递归将导致该类型的内存无限大。  

> 注意：  
>
> > 在函数中，把state结构类型变量赋值给一个局部storage类型变量时，并不会复制该对象，而仅仅将一个引用赋值给该局部变量，所以该局部变量可以写入state变量。

&emsp;&emsp;当然，在函数中你可以直接访问一个结构对象的成员，而不必将其再次赋值给一个局部变量，因为Solidity为其创建了getter。  

```javascript
campaigns[campaignID].amount = 0
```

---

- 映射(Mapping)  

&emsp;&emsp;映射与python中的字典类似但意义不同，其声明的语法如下：  

```javascript
mapping(_KeyType => _ValueType)

//_KeyType可以是任意初等型
```

&emsp;&emsp;这意味着_KeyType可以是任何内置值类型加上bytes和string类型，但是不能被定义为复杂类型(contract types, enums, mappings, structs 以及除了bytes与string之外的所有array类型)._ValueType则可以为任意类型，包括映射。  

&emsp;&emsp;你可以将映射理解为哈希表(Hash Table)，Key的值不储存在映射中，我们只用他的`keccak256`来进行索引。  

> 因此，映射没有要设置的键或值的长度或概念。  

&emsp;&emsp;映射类型具有storage的数据储存类型，因此他允许作为状态变量，或者作为storage的引用在函数中存在，或者作为library函数的参数。但是他们不能用于public函数的返回值或者参数。  

&emsp;&emsp;你可以将映射标记为public类型，并且Solidity为他创建了一个getter()接口，_KeyType将作为getter()的参数，如果_ValueType是值类型或者结构类型，那么getter将直接返回该对象，如果_ValueType是数组或映射，那么getter将返回一个包含所有_KeyType的变量，这将可以递归下去。 实例如下：  

```javascript
pragma solidity >=0.4.0 <0.6.0;

contract MappingExample {
    mapping(address => uint) public balances;

    function update(uint newBalance) public {
        balances[msg.sender] = newBalance;
    }
}

contract MappingUser {
    function f() public returns (uint) {
        MappingExample m = new MappingExample();
        m.update(100);
        return m.balances(address(this));
    }
}
```

> 映射类型是不可迭代的，但是你可以用他实现数据结构。    

---

- delete  

&emsp;&emsp;Solidity的delete与其他语言有所不同，这里的delete是将一个对象清零，你甚至可以用他来进行变量声明，`delete a`是将a初始化为0,若delete作用于动态数组则将其length变为0,若作用于静态数组则将其所有元素清零。`delete a[x]`则将清除这个单独的元素，并不会改变length和其他元素，但是这意味着数组留下了间隙，如果你打算删除数组中的元素，或许映射是更好的选择。   

&emsp;&emsp;若作用于结构，它将重新初始化结构。换言之，删除后a的值与声明a时的值相同，但需注意以下事项：  

> delete对映射无效，因此假如struct中含有映射对象，delete并不会递归执行。但是映射单独的键值关系可以被删除：  

```javascript
delete a[msg.sender];		//这将是有效的
```

> 当对象a是一个引用时，delete将不会修改其原来的值，而是直接重置a对象本身。  

&emsp;&emsp;用法如下：  

```javascript
pragma solidity >=0.4.0 <0.6.0;

contract DeleteExample {
    uint data;
    uint[] dataArray;

    function f() public {
        uint x = data;
        delete x; // sets x to 0, does not affect data
        delete data; // sets data to 0, does not affect x
        uint[] storage y = dataArray;
        delete dataArray; // this sets dataArray.length to zero, but as uint[] is a complex object, also
        // y is affected which is an alias to the storage object
        // On the other hand: "delete y" is not valid, as assignments to local variables
        // referencing storage objects can only be made from existing storage objects.
        assert(y.length == 0);
    }
}
```

### 3. 汇编基础(Solidity Addembly)  

- 指令集  

&emsp;&emsp;首先，我们来给出Solidity的指令集，这些东西有利于理解opcodes(操作码)  

&emsp;&emsp;如果opcode带有参数(从栈顶获取)，那么参数将在括号内给出。注意，在非函数样式中，参数的顺序反了过来，这很容易理解，他与C语言传参方式相同。opcode如果带有 `-` 标记，那么他将不会往栈上push一个对象(即无返回值)，如果带有*标记，代表他们比较特殊，其他没有标记的instruction将会向栈上push一个对象，这将是他们的返回值。若opcode被F，H，B或者C标记，那么他们分别是出现自 **Frontier, Homestead, Byzantium or Constantinople**.  ** Constantinople **仍然在计划中，所有被标记C的指令将导致无效或异常。  

&emsp;&emsp;在以下指令中**mem[)**表示从a开始但不包括b的memory字节，**storage[p]**表示包含在p位置的storage内容。  

> pushi与jumpdest不能被直接使用。

&emsp;&emsp;在语法中，操作码被表示为预先定义的标识符。  

| 指令(Instruction) | ||解释(解释)|
|:---|:---:|:---:|:---|
|stop|-|F|停止执行，等价于return(0,0)|
|add(x, y)||F|x + y|
|sub(x, y)||F|x - y|
|mul(x, y)||F|x * y|
|div(x, y)||F|x / y|
|sdiv(x, y)||F|x / y, 二进制补码表示的有符号数|
|mod(x, y)||F|x % y|
|smod(x, y)||F|x % y, 二进制补码表示的有符号数|
|exp(x, y)||F|x的y次幂|
|not(x)||F|~x, x的每位取非|
|lt(x, y)||F|若 x < y 则为1, 否则为0|
|gt(x, y)||F|若 x > y 则为1, 否则为0|
|slt(x, y)||F|若 x < y 则为1, 否则为0, 二进制补码表示的有符号数|
|sgt(x, y)||F|若 x > y 则为1, 否则为0, 二进制补码表示的有符号数|
|eq(x, y)||F|若 x == y 则为1, 否则为0|
|iszero(x)||F|若 x == 0 则为1, 否则为0|
|and(x, y)||F|按位将x y进行and运算|
|or(x, y)||F|按位将x y进行or运算|
|xor(x, y)||F|按位将x y进行xor运算|
|byte(x, x)||F|x的第n个字节，其中最重要的字节是第0个字节|
|shl(x, y)||C|将y逻辑左偏移x位|
|sar(x, y)||C|将y逻辑右偏移x位|
|addmod(x, y, m)||F|(x + y) % m 具有任意精度的运算|
|mulmod(x, y, m)||F|(x * y) % m 具有任意精度的运算|
|keccak256(p, n)||F|keccak(mem[p...(p+n)))|
|jump(label)|-|F|jump到 label / code 的位置|
|jumpi(lable, cond)|-|F|jump到  label 若 cond 非零|
|pc||F|当前代码位置|
|pop(x)|-|F|stack弹出一个元素|
|dup 1 ... dup 16|*|F|将第n个stack的slot复制到栈顶(从顶算起)|
|swap 1 ... swap 16|*|F|交换顶与栈底的第n个slot|
|mload(p)||F|mem[p...(p+32))|
|mstore(p, v)|-|F|mem[p...(p+32)) := v|
|mstore8(p, v)|-|F|mem[p] := v & 0xff 只修改一个字节|
|sload(p)||F|storage[p]|
|sstore(p, v)|-|F|storage[p] := v|
|msize||F|memory的大小，即最大可访问的内存索引|
|gas||F|仍可用于执行的gas的量|
|address||F|当前合约或正在执行的上下文的地址|
|balance(a)||F|地址a的账户存款,以wei表示|
|caller||F|call的sender(不包括delegatecall)|
|callvalue||F|当前发送call所发送eth的总量,以wei表示|
|calldatasize||F|以字节表示的当前call data的大小|
|calldatacopy(t, f, s)|-|F|从calldata的f位置复制s个字节到memory的t位置|
|extcodesize||F|当前合约或上下文的代码的大小|
|extcodecopy(a, t, f, s)|-|F|与codecopy(t, f, s)类似,但是是从地址a处复制|
|returndatasize||B|上次返回值的大小|
|returndatacopy(t, f, s)|-|B|将f位置的返回值复制s字节到memory的t位置|
|extcodehash(a)||C|地址a的hash|
|create(v, p , n)||F|以mem[p...(p+n))处的代码创建一个新的合约并且发送v数量的wei,返回新地址|
|call(g, a, v, in, insize,out, outsize)||F|将mem[in...(in+insize))作为输入调用a地址的合约,提供g数量的gas,若出错则返回0(gas耗尽),成功返回1|
|callcode(g, a, v, in, insize, out, outsize)||F|与call相同,但是保留当前上下文|
|delegatecall(g, a, in, insize, out, outsize)||B|与callcode相同,但是保持当前的caller与callvalue|
|staticcall(g, a, in, insize, out, outsize)||B|与call(g, a, 0, in, insize, out, outsize)相同,但是不允许状态改变|
|return(p, s)|-|F|结束执行,返回mem[p...(p+s))处的数据|
|revert(p, s)|-|B|结束执行,回滚状态的改变,返回mem[p...(p+s))处的数据|
|selfdestruct(a)|-|F|结束执行,销毁当前合约,并将全部余额打入地址a|
|invalid|-|F|以无效指令结束执行|
|origin||F|交易发送者|
|gasprice||F|交易的gas价格|
|blockhash(b)||F|块nr b的哈希-仅限于最近256个块,不包括当前块|
|coinbash||F|当前挖矿的收益|
|timestamp||F|以秒为单位的当前区块的时间戳,从创世纪开始算起|
|number||F|当前的区块数|
|difficulty||F|当前区块的困难度|
|gaslimit||F|当前区块的gas限制|

- 字面量(Literals)

&emsp;&emsp;你可以直接使用十进制或者十六进制的符号作为整数常量，并且pushi指令将会自动执行，如下代码2+3的到5然后和string "abc"进行and运算。最终结果被赋值给局部变量x。string是左对齐的并且不能超过32字节。

```javascript
assembly { let x := and("abc", add(3, 2)) }
```

- 函数风格(Functional Style)

&emsp;&emsp;对于opcode序列,通常很难看到某些opcode的实际参数是什么。如下例子中，3被加到当前memory的0x80的的位置。  

```javascript
3 0x80 mload add 0x80 mstore
```

Solidity的内联汇编有函数风格的表示，如下：

```javascript
mstore(0x80, add(mload(0x80), 3))
```

&emsp;&emsp;如果从右到左读取代码，最终得到的常量和opcode序列完全相同，但值的结束位置要清楚得多。  

&emsp;&emsp;如果您关心确切的栈布局，只需注意函数或opcode的语法第一个参数将放在栈的顶部。  

- 访问外部调用变量，函数和库  

&emsp;&emsp;你可以使用Solidity变量和其他标识符的名称来访问它们。对于存储在memory位置中的变量，他们的地址而不是值将会被推送到栈上。存储在storage位置中的变量是不同的，因为它们可能不会占用完整的存储槽，所以它们的"地址"由slot和slot内的字节偏移量组成。要检索变量x指向的插槽，可以使用x_slot，并使用x_offset字节偏移量索引。  

&emsp;&emsp;例如：  

```javascript
pragma solidity >=0.4.11 <0.6.0;

contract C {
    uint b;
    function f(uint x) public view returns (uint r) {
        assembly {
            r := mul(x, sload(b_slot)) // ignore the offset, we know it is zero
        }
    }
}
```

&emsp;&emsp;如果访问的变量的类型跨度小于256位(例如uint64、address、bytes16或byte)，则不能对不属于类型编码的位进行任何假设。尤其是，不要假设它们为零。为了安全起见，在重要的上下文中使用数据之前，请务必正确地清除数据：`uint32 x=f();assembly x：=and(x,0xffffffff) /*现在使用x*/ ` 清除签名类型，可以使用signextend的opcode。  

> 对Label的支持从0.5.0后被移除，只能使用function或者loop，而不能使用万恶的goto。  

&emsp;&emsp;你可以使用let关键字声明一个之在汇编内可见的局部变量，并且之在当前的代码块内可见。let指令将会新建一个栈的slot来存储变量并且代码块结束时自动移除。你需要为他提供一个初始化值，否则他默认为0。当然你可以按照更复杂的函数式来实现。  

```javascript
pragma solidity >=0.4.16 <0.6.0;

contract C {
    function f(uint x) public view returns (uint b) {
        assembly {
            let v := add(x, 1)
            mstore(0x80, v)
            {
                let y := add(sload(v), 1)
                b := y
            } // y is "deallocated" here
            b := add(b, v)
        } // v is "deallocated" here
    }
}
```

> 将汇编的局部变量复制给函数的局部变量是可行的，需要注意的是，在将storage或memory类型的指针复制给变量时要格外小心，你只会修改指针而不会修改变量。  
> 变量只能被赋予一个确切值，假如你要获得一个多返回值函数的返回值，那么你需要提供多个变量。  

```javascritp
{
    let v := 0
    let g := add(v, 2)
    function f() -> a, b { }
    let c, d := f()
}
```
&emsp;&emsp;if语句条件执行，但是没有"else"的部分。如果你想提供多重选择，你可以考虑使用switch语句。  

```javascript
{
    if eq(value, 0) { revert(0, 0) }
}
```

> 程序体需要大括号

&emsp;&emsp;你可以使用switch语句来实现基本的"if/else"语句，你可以使用 **default **关键字来声明一个fallback或者默认选项。

```javascript
{
    let x := 0
    switch calldataload(4)
    case 0 {
        x := calldataload(0x24)
    }
    default {
        x := calldataload(0x44)
    }
    sstore(0, div(x, 2))
}
```

> switch块不需要大括号，但是每个case需要大括号。  

&emsp;&emsp;汇编支持for风格的循环，他包含一个初始化部分，一个条件判断部分和一个迭代部分。条件判断部分必须使用函数风格，而另外两个部分则使用代码块，若初始化部分声明了某些变量，那么他们的作用与将延伸值循环体内(包括条件判断与迭代部分)。  

&emsp;&emsp;以下例子是计算一片内存的和：  

```javascript
{
    let x := 0
    for { let i := 0 } lt(i, 0x100) { i := add(i, 0x20) } {
        x := add(x, mload(i))
    }
}
```

&emsp;&emsp;当然，你也可以用他来实现while风格：  

```javascript
{
    let x := 0
    let i := 0
    for { } lt(i, 0x100) { } {     // while(i < 0x100)
        x := add(x, mload(i))
        i := add(i, 0x20)
    }
}
```

&emsp;&emsp;汇编也支持低级函数的定义，参数和返回地址来自栈，返回值也将布置到栈上，调用一个函数看起来像执行一段函数类型opcode。函数可以被定义在任意位置，其可视范围是所定义的代码块，在函数体内你不能访问外部的变量，而且函数没有显示的return语句。  

> 如果你的函数有多个返回值，那么你需要将他们复制给一个元组(即多个变量)  

&emsp;&emsp;下面的示例通过平方和乘法实现幂函数：  

```javascript
{
    function power(base, exponent) -> result {
        switch exponent
        case 0 { result := 1 }
        case 1 { result := base }
        default {
            result := power(mul(base, base), div(exponent, 2))
            switch mod(exponent, 2)
                case 1 { result := mul(base, result) }
        }
    }
}
```

- Solidity 中的转换(Conventions in Solidity)  

&emsp;&emsp;与EVM汇编相比，Solidity有些类型不足256位，为了使计算更有效，EVM通常将他们以256为来对待，强行把它们放在一个slot内，而高阶位元只在必要时才会被清理，就在它们被写入内存或执行比较之前不久。所以如果你想要使用内联汇编来访问他们的话，你必须手动清零高位。  
&emsp;&emsp;solidity以一种非常简单的方式管理内存：内存中的位置0x40处有一个“空闲内存指针”。如果要分配内存，只需使用从指针指向的位置开始的内存，并相应地更新它。但我们无法保证内存之前没有被使用过，所以你不能假设他的初始内容是0。没有内置的内存释放或回收机制，下面是一个内存分配的例子：    

```javascript
function allocate(length) -> pos {
  pos := mload(0x40)
  mstore(0x40, add(pos, length))
}
```

&emsp;&emsp;前64个字节的内存可以用作短期分配的“临时空间”。储存空闲内存指针后的32个字节(即从0x60开始)应永久为零，并用作空动态内存数组的初始值。这意味着可分配内存从0x80开始，这是可用内存指针的初始值。    

&emsp;&emsp;solidity中memory数组中的元素总是占用32字节的倍数(是的，对于byte[]甚至是这样，但对于bytes和string则不是这样)。多维memory数组是指向memory数组的指针。动态数组的长度存储在数组的第一个solt(第一个32byte)中，然后是数组元素。  

> 静态大小的memory数组没有长度字段，但日后可能会添加该字段，以便在静态大小的数组和动态大小的数组之间实现更好的可转换性，因此请不要依赖于此。  

- 独立汇编(Standalone Assembly)  

&esmp;&emsp;独立汇编是区块链逆向的基础，我们直接来感受一下吧：  

```javascript
pragma solidity >=0.4.16 <0.6.0;

contract C {
  function f(uint x) public pure returns (uint y) {
    y = 1;
    for (uint i = 0; i < x; i++)
      y = 2 * y;
  }
}
```

&emsp;&emsp;对应汇编如下：  

```assembly
{
  mstore(0x40, 0x80)	 // 储存    "空闲memory指针”
  
  // 函数调度器，注意这里的运行方式
  switch div(calldataload(0), exp(2, 226))
  case 0xb3de648b {
    let r := f(calldataload(4))
    let ret := $allocate(0x20)
    mstore(ret, r)
    return(ret, 0x20)
  }
  default { revert(0, 0) }
  // 内存分配器
  function $allocate(size) -> pos {
    pos := mload(0x40)
    mstore(0x40, add(pos, size))
  }
  // 合约函数部分
  function f(x) -> y {
    y := 1
    for { let i := 0 } lt(i, x) { i := add(i, 1) } {
      y := mul(2, y)
    }
  }
}
```
> 注意
>
> > 函数的调度是通过一个16位字节码来实现的，即上例的switch/case部分，switch的操作就是将调用地址转化成一个16位字节码，若调用地址与某函数字节码对应，则调至该函数，这一步分在区块链逆向中十分常见。  

&emsp;&emsp;关于汇编的语法我不想多将，其更重要的是EVM的运行机制，我们会在后面进行说明。



### 4. 杂项(Miscellaneous)  

- 状态变量的内存布局  



















---  

<h5 align=center><a href="https://dyf.netlify.com">Author dyf@Ever404</a><h5>






