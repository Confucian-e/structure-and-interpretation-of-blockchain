&emsp;&emsp;`RLP（Recursive Length Prefix）`递归长度前缀编码，`RLP`主要用于以太坊中数据的网络传输和持久化存储。

### 为什么需要`RLP`编码
&emsp;&emsp;比较常见的序列化方法有`JSON`，`ProtoBuf`，但是这些序列化方法在以太坊这样的场景下都有一些问题。

&emsp;&emsp;比如像`Json`编码，编码后的体积比较大。
```golang
type Persion struct {
    Name string
    Age uint 
}
 p := &Persion{Name: "Tom", Age: 22}
 data, _ := json.Marshal(p)
 fmt.Println(string(data))
//  {"Name":"Tom","Age":22}
```
&emsp;&emsp;从编码后的结果可以看到，其实我们需要的数据是`Name`, `Tom`, `Age`, `22`，其余的括号和引号都是描述这种格式的，也就是冗余数据。

&emsp;&emsp;当然，会有人说可以用`protoBuf`这样的二进制格式，但是例如`JavaScript`这样的弱类型语言，是没有`int`类型的，所有数字都是用`Number`类型，底层用浮点数来实现，这样就会导致因为语言的不同编码后的数据有一定差异，最后得到的hash值也不同。

&emsp;&emsp;针对这些问题，以太坊设计了`RLP`编码，同时也大幅简化了编码规则。
### `RLP`编码定义
`RLP`简化了编码的类型，只定义了两种类型编码：
- byte数组
- byte数组的数组，也就是列表

### `RLP`编码基于上面两种数据类型提出了5条编码规则;

##### 规则一
&emsp;&emsp;对于值在[0, 127]之间的单个字节，其编码是其本身；
```
a的编码是97
```
##### 规则二
&emsp;&emsp;如果byte数组的长度`l<=55`，编码的结果是数组本身，再加上`128+l`作为前缀。
```
空字符串的编码是`128`，即 `128=128+0`
`abc`的编码是`131 97 98 99`，其实`131=128+len("abc")`, `97 98 99`依次是`a b c`
```
##### 规则三
&emsp;&emsp;如果数组长度大于`55`，编码结果第一个值是`183（128+55）`加数组长度的编码的长度，然后是数组长度本身的编码，最后是byte数组的编码。
```
编码一个重复1024次"a"的字符串，其结果是`185 4 0 97 97 97 ...` 
```
&emsp;&emsp;1024按照`大端编码`是`0000 0000 001`转换为十进制是`0 0 4 0`，省略前面的`0`,长度为2， 因此`185 = 183 + 2`

##### 规则四
&emsp;&emsp;如果列表长度小于55，编码结果第一位是192加列表长度的编码的长度，然后依次连接各个子列表的编码。
```
`["abc", "def"]`的编码结果是`200 131 97 98 99 131 100 101 102`
```
&emsp;&emsp;其中`abc`的编码是`131 97 98 99`, `131 = 128 + l`长度是4，`def`的编码是`131 100 101 102`，长度是4，总长度为`8`，编码结果的第一位`200 = 192 + 8`

##### 规则五
&emsp;&emsp;如果列表的长度超过55，编码结果第一位是`247(192 + 55)`加列表长度的编码长度，然后是列表本身的编码，最后依次连接子列表的编码
```
["The length of this sentence is more than 55 bytes, ", "I know it because I pre-designed it"]  
```
的编码结果是:

```

248 88 179 84 104 101 32 108 101 110 103 116 104 32 111 102 32 116 104 105 115 32 115 101 110116 101 110 99 101 32 105 115 32 109 111 114 101 32 116 104 97 110 32 53 53 32 98 121 116 101115 44 32 163 73 32 107 110 111 119 32 105 116 32 98 101 99 97 117 115 101 32 73 32 112 114101 45 100 101 115 105 103 110 101 100 32 105 116
```

&emsp;&emsp;其中前两个字节的计算方式如下：

- `248 = 247 + 1`

- `88 = 86 + 2`， 在规则3的示例中，长度为86，而在此例中，由于有两个子字符串，每个子字符串本身的长度的编码各占1字节，因此总共占2字节。

- 第3个字节`179`依据规则2得出

第55个字节`163`同样依据规则2得出`163=128+35`



> 其中`规则三`，`规则四`，`规则5`是递归定义的，就是可以允许嵌套
