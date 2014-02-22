[TOC]

## 判定字节顺序

`os`核心模块提供了一个方法`endianness()`,用于判断当前机器的字节顺序.`endianness()`是一个无参数函数,返回一个字符串表示当前机器的字节顺序. 如果当前机器为大端字节顺序`endianness()`返回"`BE`", 如果为小端字节顺序则放回`LE`.清单8-5 中的实例调用`endianness()`并在控制台中打印出结果

清单 8-5. 使用`os.endianness()`方法判定一个机器的字节顺序

```javascript
var os = require("os");
console.log(os.endianness());
```
    
## 类型化的数组规范(Typed Array Specification)

在阐述Node中的二进制数据处理方式之前,先来了解一下标准的Javascript是如何处理二进制数据的.这种方式被成为Typed Array Specification, 和普通的JavaScript变量不同,二进制数据数组有一个在运行时不能改变的类型,因为Typed Array Specification作为JavaScript语言的一部分, 它是能在大多数浏览器中工作的, 具体要看浏览器是否实现了这个特性.

### ArrayBuffers

JavaScript的二进制数据API包括两部分: 一个Buffer, 以及一个View, Buffer是通过`ArrayBuffer`数据类型来实现的, 它是一个通用容器, 用来存储字节数组. ArrayBuffer是一个固定长度的结构, 一旦创建便不能再修改它的大小. ArrayBuffer通过构造函数ArrayBuffer()创建. 下面的示例创建了一个**内存空间占用**为1024字节的ArrayBuffer

清单 8-6 创建一个1024字节的`ArrayBuffer`
```javascript
var buffer = new ArrayBuffer(1024);
```
    
ArrayBuffer的工作方式类似与普通的Javascript数组.使用数组下标读写单个字节.因为ArrayBuffer不能修改大小,向**不存在的数组索引**写数据不会实际修改底层的数据结构, 这个错误会被JavaScript引擎自动忽略, 来看下面的代码

代码 8-7: 写入ArrayBuffer并输出结果

```javascript
var buffer = ArrayBuffer(2);
buffer[0] = 0x01;
buffer[1] = 0x02;
buffer[2] = 0x03
```

这段代码创建了一个长度为2字节的二进制数据变量buffer, 我向buffer[2]写入了一个值0x03, 下面我们在控制台中输出这个buffer的值.

代码 8-8: 显示代码8-7的结果

```javascript
console.log(buffer);
{ '0': 1, '1': 2, slice: [Function: slice], byteLength: 2 }
```

在控制台输出中,我们注意到`byteLength`为2, 它表示ArrayBuffer的字节长度, 这个值在ArrayBuffer创建时就被固定不能改变了,类似与数组的length属性, byteLength可以用于遍历ArrayBuffer的内容


清单 8-9: 用byteLength属性遍历显示ArrayBuffer的每一个字节
    
```javascript
var foo = new ArrayBuffer(4);
foo[0] = 0;
foo[1] = 1;
foo[2] = 2;
foo[3] = 3;
for (var i = 0, len = foo.byteLength; i < len; i++) {
    console.log(foo[i]);
}
```
    
#### Slice()

可以使用`ArrayBuffer.slice(int start, int end)`方法抽取ArrayBuffer的内容拷贝一个新的ArrayBuffer, slice()方法有两个参数:起始位置(包括)和结束位置(不包括), 这两个参数定义了要拷贝的范围

清单 8-10: 使用`slice()`方法创建一个全新的ArrayBuffer对象

```javascript
var foo = new ArrayBuffer(4);
foo[0] = 0;
foo[1] = 1;
foo[2] = 2;
foo[3] = 3;
console.log(foo.slice(2, 4));
console.log(foo.slice(2, foo.byteLength));
console.log(foo.slice(2));
console.log(foo.slice(-2));
// 上面四种调用方式获得相同的结果: [2,3]
```

要注意的是slice()方法返回的是原始数据的一个拷贝.因此修改slide()返回的buffer对象**不会**修改原始数据, 这与Node中Buffer对象的实现方式不同[^1], 请参考清单8-11.

清单 8-11: 使用**slice()**方法创建一个全新的ArrayBuffer对象

```javascript
var bytes = new ArrayBuffer(4);
var bar;
bytes[0] = 0;
bytes[1] = 1;
bytes[2] = 2;
bytes[3] = 3;
// 创建一个foo的拷贝,并修改它
bar = bytes.slice(0);
bar[0] = 0xc;
console.log(foo);
console.log(bar);
```
名为foo的ArrayBuffer被创建并填充数据. 然后复制整个foo的内容到bar,然后修改bar的第一个字节,最后在控制台中输出foo和bar的内容,清单8-12显示了输出结果, 注意这两个buffer第一个字节的差异.

清单 8-12: 运行清单8-11的输出

```javascript  
{ '0': 0,
  '1': 1,
  '2': 2,
  '3': 3,
  slice: [Function: slice],
  byteLength: 4 }
{ '0': 12,
  '1': 1,
  '2': 2,
  '3': 3,
  slice: [Function: slice],
  byteLength: 4 }
```  

### ArrayBuffer Views

直接处理字节数组是一个大麻烦. 在`ArrayBuffer`上添加一个抽象层(这个抽象层被称作View)来操作数据, 下面给出一个示例来删除View是如何工作的


表格 8-1. JavaScript’的个中ArrayBuffer View描述

View Type         | Element Size (Bytes) | Description
------------------| -------------------- | -----------
Int8Array         | 1                    | Array of 8-bit signed integers.
Uint8Array        | 1                    | Array of 8-bit unsigned integers.
Uint8ClampedArray | 1                    | Array of 8-bit unsigned integers. Values are clamped to be in the 0–255 range.
Int16Array        | 2                    | Array of 16-bit signed integers.
Uint16Array       | 2                    | Array of 16-bit unsigned integers.
Int32Array        | 4                    | Array of 32-bit signed integers.
Uint32Array       | 4                    | Array of 32-bit unsigned integers.
Float32Array      | 4                    | Array of 32-bit IEEE floating point numbers.
Float64Array      | 8                    | Array of 64-bit IEEE floating-point numbers.

**Note**

虽然 `Uint8Array` 和 `Uint8ClampedArray` 非常相似,但却有一个关键的差异,这个差异表现在超过0-255这个值域时如何处理. `UInt8Array` 在处理超过255的值时: 255, 256, 257 被解释为 255, 0, 1. `Uint8ClampedArray` 的行为则不同,任何大于255的值被重置为255,任何小于0的值被重置为0, 那么对于 `Uint8ClampedArray` : 255, 256, 257 被解释为 255, 255,255.


清单 8-13: `Uint32Array` 示例

```javascript  
var buf = new ArrayBuffer(8);
var view = new Uint32Array(buf);
view[0] = 100;
view[1] = 256;
console.log(view);
```

清单8-14显示了输出结果(我对结果作了一下格式化,使代码更加工整一些),头两行显示了写入到View中的值,100,256.接下来是一个`BYTES_PER_ELEMENT`属性,这是一个只读属性,包含在每个View对象中,表示View中的每个元素的原始字节长度,紧接着`BYTES_PER_ELEMENT`的是几个View对象的方法.

```javascript  
{ 
    '0': 100,
    '1': 256,
    BYTES_PER_ELEMENT: 4,
    get: [Function: get],
    set: [Function: set],
    slice: [Function: slice],
    subarray: [Function: subarray],
    buffer:{ 
        '0': 100,
        '1': 0,
        '2': 0,
        '3': 0,
        '4': 0,
        '5': 1,
        '6': 0,
        '7': 0,
        slice: [Function: slice],
        byteLength: 8 
    },
    length: 2,
    byteOffset: 0,
    byteLength: 8 
}
```    

底层的ArrayBuffer对象被显示为buffer属性.在这个例子中0-3字节表示100,4-7字节表示256.

**注意**:-- 

**256** 等同于 **2<sup>8</sup>**, 它不能用单个字节表示. 单个字节都能够表示的最大值为 **255**, 因此256的16进制表示为 **01 00**

和`ArrayBuffer`的`slice()`方法不同的是, slice()方法返回的是一个对原始ArrayBuffer数据的一个全新拷贝, 而View则是直接操作底层的`ArrayBuffer`对象.修改View的值会直接修改底层ArrayBuffer对象的值.Also, two views having the same ArrayBuffer can accidentally (or intentionally) change each other’s values.清单8-15中的示例所示: 一个包含4字节的ArrayBuffer,由一个`Uint32Array`的View和一个`Uint8Array`的View共享.首先写入值 **100** 到`Uint32Array`,然后输出,再写入值 **1** 到`Uint8Array`的第二个字节,再次输出`Uint32Array`:

清单 8-15: View之间的相互影响

```javascript
var buf = new ArrayBuffer(4);
var view1 = new Uint32Array(buf);
var view2 = new Uint8Array(buf);
// write to view1 and print the value
view1[0] = 100;
console.log("Uint32 = " + view1[0]);
// write to view2 and print view1's value
view2[1] = 1;
console.log("Uint32 = " + view1[0]);
```

清单 8-16. 运行清单 8-15 的输出结果, 果不其然, 第一个print语句显示的值为**100**, 第二条语句执行时, 该值增加到 **356**, 在此例中, 为了演示这种副作用,这个行为是我们所期望的, 但是在更加复杂的应用程序中, 当在`ArrayBuffer`上创建多个`View`的时候,必须谨小慎微.

```javascript
Uint32 = 100
Uint32 = 356
```
#### View对象的 **BYTES_PER_ELEMENT** 属性和 **ArrayBuffer.byteLength** 的关系

`View`只能在长度为`BYTES_PER_ELEMENT`属性的整数倍的`ArrayBuffer`对象上创建. 比如,一个4字节的ArrayBuffer对象能够用于构造一个`Int32Array`的View, 用于存储单个整数. 但是同样的4字节buffer不能用于构造一个元素为8字节长的`Float64Array`.

#### 构造函数

~~Each type of view has four constructors. One form, which you’ve already seen, takes an `ArrayBuffer` as its first argument. This constructor function can also optionally specify both a starting byte offset in the ArrayBuffer and the view’s length. The byte offset defaults to 0 and __*must*__ be a multiple of BYTES_PER_ELEMENT, or else a `RangeError` exception is thrown. If omitted, the length will try to consume the entire ArrayBuffer, starting at the byte offset. These arguments, if specified, allow the view to be based on a piece of the ArrayBuffer instead of the entire thing. This is especially useful if the ArrayBuffer length is not an exact multiple of the view’s BYTES_PER_ELEMENT. In the example in Listing 8-17, which shows how a view can be constructed from a buffer whose size is not an exact multiple of `BYTES_PER_ELEMENT`, an `Int32Array` view is built on a 5-byte ArrayBuffer. The byte offset of 0 indicates that the view should begin at the first byte of the ArrayBuffer. Meanwhile, the length argument specifies that the view should contain a single integer. Without these arguments, it would not be possible to construct the view from this ArrayBuffer. Also, notice that the example contains a write to the byte at `buf[4]`. Since the view uses only the first four bytes, this write to the fifth byte does not alter the data in the view.~~

清单 8-17. 构造一个基于ArrayBuffer部分数据的View 

```javascript`
var buf = new ArrayBuffer(5);
var view = new Int32Array(buf, 0, 1);
view[0] = 256;
buf[4] = 5;
console.log(view[0]);
```
#### 创建一个空View对象

第二种构造函数用于创建一个预定义长度为 **n** 的空View对象,这个形式的构造函数还创建了一个足够大的`ArrayBuffer`对象来容纳 **n** 个View元素. 例如, 清单8-18创建了一个空的`Float32Array` View来容纳2个浮点数. 在这个构造函数的幕后,还创建了一个长度为8字节的`ArrayBuffer`对象来容纳这两个浮点数. 在构造过程中, `ArrayBuffer`的所有字节被初始化为 **0**.

清单 8-18. 创建一个空的 **Float32Array** View对象
```javascript`
var view = new Float32Array(2);
```

#### 从数据值创建View对象

第三种形式的构造函数接受一个值数组用于填充View.数组中的值被转换为核实的数据类型,并存储在View中,此类构造函数还创建了一个全新的`ArrayBuffer`对象来容纳这些值.清单8-19显示了一个示例,用值 **1,2,3** 来填充一个`Uint16Array`

清单 8-19. 从一个包含3个值的数组创建一个`Uint16Array`对象
```javascript`
var view = new Uint16Array([1, 2, 3]);
```

#### 从其他View对象创建一个新的View对象

~~The fourth version of the constructor is very similar to the third. The only difference is that instead of passing in a standard array, this version accepts another view as its only argument. The newly created view also instantiates its own backing ArrayBuffer—that is, the underlying data is not shared. Listing 8-20 shows how this version of the constructor is used in practice. In this example, a 4-byte ArrayBuffer is used to create an Int8Array view containing four numbers.The Int8Array view is then used to create a new Uint32Array view. The Uint32Array also contains four numbers,corresponding to the data in the Int8Array view. However, its underlying ArrayBuffer is 16 bytes long instead of 4.Of course, because the two views have different ArrayBuffers, updating one view does not affect the other.~~

清单 8-20. 从 **Int8Array** 创建一个 **Uint32Array**

```javascript`
var buf = new ArrayBuffer(4);
var view1 = new Int8Array(buf);
var view2 = new Uint32Array(view1);
console.log(buf.byteLength); // 4
console.log(view1.byteLength); // 4
console.log(view2.byteLength); // 16
```

[^1]: Node中Buffer类型的slice()方法和JavaScript的ArrayBuffer类型的slice方法在语义上有很大的区别,Node中Buffer类型的slide()返回的是一个引用(类似指针),对其修改将会同时修改原始的Buffer对象, 而ArrayBuffer通过slice()方法调用创建的是一个全新的ArrayBuffer对象,此对象拥有独立的内存空间,使用的时候请特别注意这两者的区别.
