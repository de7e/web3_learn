# 理解ERC165标准
## 一.理解ERC165

  最近打算认真学习下ERC1155，但发现ERC777、ERC1820、ERC165很难绕过去，所以本着扎实的想法，那就从ERC165开始吧。

  所以本文主要是围绕着EIP165提案，结合我对其他人学习资料的学习，希望能够以简单易懂、准确，容易理解的方式，分享和记录我对ERC165标准的理解。


### ERC165是什么？
  1.一种接口检测查询和发布标准
  2.更具体的说是**一种可以查询目标合约是否支持了特定的接口**，当然同时**你也可以发布自己实现的接口，以供他人查询和交互**。
  
### ERC165有什么用？
**可以通过其确定一个合约支持哪些接口，这样就知道如何与它进行交互。**

例子：
  如对于NFT市场来说，就可以通过ERC165标准来查询你的智能合约是不是实现了ERC721接口，如果是的话就可以使用ERC721的各种方法与你进行交互和操作。
	
  **所以ERC165提供了一种确定性的互操作支持。**



## 二.ERC165如何实现注册和查询接口？
  根据官方定义
>   一个符合ERC-165规范的合约需要实现如下接口：

```
interface ERC165 {
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}
```
  
  1.任何人都可以根据这个方法，加上 要查询的“**接口标识符(interfaceId)**”来查询任意合约是否支持指定的接口。
  
  2.实现了则返回true，未实现则返回false。这样就有了一种清晰和确定性的方式能够知道目标合约是否支持哪些具体的接口与它交互。
  
  
 
 那么这里**接口标识符(interface)** 又是什么呢？
  根据提案解释
> 将接口标识符(interfaceId)定义为接口中所有函数选择器的异或。

**下面代码示例显示了如何计算接口标识符**

```
pragma solidity ^0.4.20;

interface Solidity101 {
    function hello() external pure;
    function world(int) external pure;
}

contract Selector {
    function calculateSelector() public pure returns (bytes4) {
        Solidity101 i;
        return i.hello.selector ^ i.world.selector;
    }
}
```

**简单的说就是一个接口，无论定义了多少个方法，这个interfaceId就是其所有方法(函数选择器)的异或结果。**

  至于为什么使用异或，使用异或是因为这是一种很方便结合得到一个接口中所有方法选择器的唯一标识表示方式。 ps: 其实可能会有异或函数选择器重叠的概率，但非常小。

  **所以从这里可以得到关于interfaceId的几个关键信息**
- interfaceId并不是单个函数的选择器，而是整个接口的所有函数选择器的共同异或结果。
- 在solidity中，你可以使用 type(yourInterface).interfaceId 来获得任何接口的接口ID(因为solidity语言适配了使用ERC165方式计算得到interfaceId的支持)
- 所以对于固定常用的接口，接口ID是固定的。比如ERC165的接口ID就是0x01ffc9a7


**到这里就可以简单总结一下对于ERC165标准的理解：**
`只要一个合约内实现了一个名为"supportsInterface的方法"，并能够根据你查询的接口ID(interfaceId)是否存在，而返回true 或 false ，即这个合约就算实现了ERC165。`

## 三.具体应用
### **1.怎样识别合约是否支持ERC165？**
  **需要2步来检测**
  

```solidity
//1.使用 staticall发送
0x01ffc9a701ffc9a700000000000000000000000000000000000000000000000000000000

//其实就等同于调用 
contract.supportsInterface(0x01ffc9a7)
```
  **并预期合约返回true**
  
  ``` 
//2.使用 staticall发送0x01ffc9a7ffffffff00000000000000000000000000000000000000000000000000000000

//其实就等同于调用 
contract.supportsInterface(0xffffffff)
  ```
  **并预期合约返回false**
  
  ----
  **那么为什么要发送两次呢？**
  
   第一次是为了确定合约实现了ERC165的supportsInterface方法，所以会返回true。
   
  第二次是为了确定合约正确实现了此方法，这里有两个原因：
> 因为0xffffffff(换算成二进制就是32位个全1)这个值是几乎不太可能是一个正常的接口ID
>     所以
>     1.确保你的实现是正确的，不会有不存在的interface你也返回true。
>     2.尤其是对于在erc165标准出现之前的合约，预期需要确定性的返回false。

Tips:实际上使用中不需要staticall，直接调用合约的supportsInterface()方法传参接口ID即可。


###  **2.如何检测一个已部署的合约是否支持指定的接口？**
  如同识别ERC165接口，只需要传入对于的interfaceId
solidity代码:
```
//通过type(yourInterface).interfaceId 得到你要查询的接口ID 如 0x80ac58cd (ERC721的接口ID)

//调用查询
IERC165(token).supportsInterface(0x80ac58cd);
```

### **3.实现和发布自己的ERC165接口**

  要让自己的合约支持ERC165接口查询，本质上只要在合约内实现了"supportsInterface"，并能根据接口ID是否与否正确返回true 或 false 即可。

但有两种不同的实现方式：
- 1种是纯函数直接返回，
- 另一种是把接口ID通过mapping来进行保存。

`Tip：
前者的优势是若注册的接口数量少于3个，gas成本更低。后者优势是若注册的接口数量为3个或以上，则成本更低。`


**下面为两种官方提案的ERC165实现方式代码**

**1.以mapping的方式注册**
<!--StartFragment-->

```
pragma solidity ^0.4.20;

import "./ERC165.sol";

contract ERC165MappingImplementation is ERC165 {
    /// @dev You must not set element 0xffffffff to true
    mapping(bytes4 => bool) internal supportedInterfaces;

    function ERC165MappingImplementation() internal {
        supportedInterfaces[this.supportsInterface.selector] = true;
    }

    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return supportedInterfaces[interfaceID];
    }
}

interface Simpson {
    function is2D() external returns (bool);
    function skinColor() external returns (string);
}

contract Lisa is ERC165MappingImplementation, Simpson {
    function Lisa() public {
        supportedInterfaces[this.is2D.selector ^ this.skinColor.selector] = true;
    }

    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```

<!--EndFragment-->



**2.以纯函数的方式直接返回**
<!--StartFragment-->

```
pragma solidity ^0.4.20;

import "./ERC165.sol";

interface Simpson {
    function is2D() external returns (bool);
    function skinColor() external returns (string);
}

contract Homer is ERC165, Simpson {
    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return
          interfaceID == this.supportsInterface.selector || // ERC165
          interfaceID == this.is2D.selector
                         ^ this.skinColor.selector; // Simpson
    }

    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```

<!--EndFragment-->

**Tip：** ERC165规范supportsInterface()的实现Gas消耗不要超过30000，以获得统一的可预期的交互保障。



## ● 总结与参考
ERC-165标准为智能合约的开发提供了重要的互操作性支持，同时也看到了一个提案或标准，往往是很多贡献者的长期点滴的构筑，最后才让这个提案更加成熟。

---
参考来源：
EIP165官方提案：https://eips.ethereum.org/EIPS/eip-165
Github EIP165提案历史讨论：https://github.com/ethereum/EIPs/issues/165
其他ERC165优质学习笔记：https://www.cnblogs.com/wanghui-garcia/p/9507128.html
