<style>
table th {
    width:200px;
}
</style>
[TOC]

## 判定字节顺序

`os` 核心模块提供了一个方法 `endianness()`, 用于判断当前机器的字节顺序.`endianness()`是一个无参数函数,返回一个字符串表示当前机器的字节顺序. 如果当前机器为大端字节顺序 `endianness()` 返回 **BE**, 如果为小端字节顺序则返回 **LE**. [清单8-5](#listing-8-5) 中的示例调用 `endianness()` 并在控制台中打印出结果

清单 8-5: 使用 `os.endianness()` 方法判定一个机器的字节顺序

```javascript {#listing-8-5}
var os = require("os");
console.log(os.endianness());
```
    
## 8.2 类型化的数组规范(Typed Array Specification)

在阐述Node中的二进制数据处理方式之前,先来了解一下标准的Javascript是如何处理二进制数据的. 这种方式被成为Typed Array Specification, 和普通的JavaScript变量不同,二进制数据数组有一个在运行时不能改变的类型, 因为Typed Array Specification作为JavaScript语言的一部分, 它是能在大多数浏览器中工作的, 具体要看浏览器是否实现了这个特性.

### 8.2.1 ArrayBuffers

JavaScript的二进制数据API包括两部分: 一个Buffer, 以及一个View, Buffer是通过 `ArrayBuffer` 数据类型来实现的, 它是一个通用容器, 用来存储字节数组. `ArrayBuffer` 是一个固定长度的结构, 一旦创建便不能再修改它的大小. `ArrayBuffer` 通过构造函数 `ArrayBuffer()` 创建. [清单8-6](#listing-8-6) 中的示例创建了一个 **内存空间占用** 为 **1024** 字节的 `ArrayBuffer`

清单 8-6: 创建一个1024字节的`ArrayBuffer`

```javascript {#listing-8-6}
var buffer = new ArrayBuffer(1024);
```
    
`ArrayBuffer` 的工作方式类似与普通的Javascript数组.使用数组下标读写单个字节.因为 `ArrayBuffer` 不能修改大小, 向 **不存在的数组索引** 写数据不会实际修改底层的数据结构, 这个错误会被JavaScript引擎自动忽略, 来看下面 [清单8-7](#listing-8-7) 中的代码

清单 8-7: 写入ArrayBuffer并输出结果

```javascript {#listing-8-7}
var buffer = ArrayBuffer(2);
buffer[0] = 0x01;
buffer[1] = 0x02;
buffer[2] = 0x03
```

这段代码创建了一个长度为2字节的二进制数据变量buffer, 我向buffer[2]写入了一个值0x03, 下面我们在控制台中输出这个buffer的值.

清单 8-8: 显示代码8-7的结果

```javascript {#listing-8-8}
console.log(buffer);
{ '0': 1, '1': 2, slice: [Function: slice], byteLength: 2 }
```

在控制台输出中,我们注意到`byteLength`为2, 它表示ArrayBuffer的字节长度, 这个值在ArrayBuffer创建时就被固定不能改变了,类似于数组的length属性, byteLength可以用于遍历ArrayBuffer的内容


清单 8-9: 用byteLength属性遍历ArrayBuffer的每一个字节
    
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
    
#### 8.2.1.1 Slice()

可以使用`ArrayBuffer.slice(int start, int end)`方法从ArrayBuffer拷贝一个新的ArrayBuffer, slice()方法有两个参数:起始位置(包括)和结束位置(不包括), 这两个参数定义了要拷贝的范围

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

要注意的是slice()方法返回的是原始数据的一个拷贝.因此修改slide()返回的buffer对象 **不会** 修改原始数据, 这与Node中Buffer对象的实现方式不同[^1], 请参考 [清单8-11](#listing-8-11).

清单 8-11: 使用**slice()**方法创建一个全新的ArrayBuffer对象

```javascript {#listing-8-11}
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

清单 8-12: 运行[清单8-11](#listing-8-11)的输出

```javascript {#listing-8-11}
{ 
    '0': 0,
    '1': 1,
    '2': 2,
    '3': 3,
    slice: [Function: slice],
    byteLength: 4 
}
{ 
    '0': 12,
    '1': 1,
    '2': 2,
    '3': 3,
    slice: [Function: slice],
    byteLength: 4 
}
```  

### ArrayBuffer Views

直接处理字节数组是一个大麻烦. 在`ArrayBuffer`上添加一个抽象层(这个抽象层被称作View)来操作数据, [表8-1](#table-8-1)给出一个示例来删除View是如何工作的


表格 8-1: JavaScript’的个中ArrayBuffer View描述 

<span id="table-8-1"></span>
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
{#table-8-1}


**Note**

虽然 `Uint8Array` 和 `Uint8ClampedArray` 非常相似,但却有一个关键的差异,这个差异表现在超过0-255这个值域时如何处理. `UInt8Array` 在处理超过255的值时: 255, 256, 257 被解释为 255, 0, 1. `Uint8ClampedArray` 的行为则不同,任何大于255的值被重置为255,任何小于0的值被重置为0, 那么对于 `Uint8ClampedArray` : 255, 256, 257 被解释为 255, 255,255.


清单 8-13: `Uint32Array` 示例

```javascript {#listing-8-13}
var buf = new ArrayBuffer(8);
var view = new Uint32Array(buf);
view[0] = 100;
view[1] = 256;
console.log(view);
```

[清单8-14](#listing-8-14) 显示了输出结果(我对结果作了一下格式化,使代码更加工整一些),头两行显示了写入到View中的值,100,256.接下来是一个`BYTES_PER_ELEMENT`属性,这是一个只读属性,包含在每个View对象中,表示View中的每个元素的原始字节长度,紧接着`BYTES_PER_ELEMENT`的是几个View对象的方法.

清单 8-14: 运行[清单8-13](#listing-8-13)的输出

```javascript {#listing-8-14}
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

**注意**:

**256** 等同于 **2<sup>8</sup>**, 它不能用单个字节表示. 单个字节都能够表示的最大值为 **255**, 因此256的16进制表示为 **01 00**

和`ArrayBuffer`的`slice()`方法不同的是, slice()方法返回的是一个对原始ArrayBuffer数据的一个全新拷贝, 而View则是直接操作底层的`ArrayBuffer`对象.修改View的值会直接修改底层ArrayBuffer对象的值.Also, two views having the same ArrayBuffer can accidentally (or intentionally) change each other’s values.[清单8-15](#listing-8-15)中的示例所示: 一个包含4字节的ArrayBuffer,由一个`Uint32Array`的View和一个`Uint8Array`的View共享.首先写入值 **100** 到`Uint32Array`,然后输出,再写入值 **1** 到`Uint8Array`的第二个字节,再次输出`Uint32Array`:

清单 8-15: View之间的相互影响

```javascript {#listing-8-15}
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

[清单8-16](#listing-8-16). 运行清单 8-15 的输出结果, 果不其然, 第一个print语句显示的值为**100**, 第二条语句执行时, 该值增加到 **356**, 在此例中, 为了演示这种副作用,这个行为是我们所期望的, 但是在更加复杂的应用程序中, 当在`ArrayBuffer`上创建多个`View`的时候,必须谨小慎微.

清单 8-16. 运行[清单8-15](#listing-8-15)中代码的输出

```javascript {#listing-8-16}
Uint32 = 100
Uint32 = 356
```
#### View对象的 **BYTES_PER_ELEMENT** 属性和 **ArrayBuffer.byteLength** 的关系

`View`只能在长度为`BYTES_PER_ELEMENT`属性的整数倍的`ArrayBuffer`对象上创建. 比如,一个4字节的ArrayBuffer对象能够用于构造一个`Int32Array`的View, 用于存储单个整数. 但是同样的4字节buffer不能用于构造一个元素为8字节长的`Float64Array`.

#### 构造函数

~~Each type of view has four constructors. One form, which you’ve already seen, takes an `ArrayBuffer` as its first argument. This constructor function can also optionally specify both a starting byte offset in the ArrayBuffer and the view’s length. The byte offset defaults to 0 and __*must*__ be a multiple of BYTES_PER_ELEMENT, or else a `RangeError` exception is thrown. If omitted, the length will try to consume the entire ArrayBuffer, starting at the byte offset. These arguments, if specified, allow the view to be based on a piece of the ArrayBuffer instead of the entire thing. This is especially useful if the ArrayBuffer length is not an exact multiple of the view’s BYTES_PER_ELEMENT. In the example in Listing 8-17, which shows how a view can be constructed from a buffer whose size is not an exact multiple of `BYTES_PER_ELEMENT`, an `Int32Array` view is built on a 5-byte ArrayBuffer. The byte offset of 0 indicates that the view should begin at the first byte of the ArrayBuffer. Meanwhile, the length argument specifies that the view should contain a single integer. Without these arguments, it would not be possible to construct the view from this ArrayBuffer. Also, notice that the example contains a write to the byte at `buf[4]`. Since the view uses only the first four bytes, this write to the fifth byte does not alter the data in the view.~~

清单 8-17. 构造一个基于ArrayBuffer部分数据的View 

```javascript` {#listing-8-17}
var buf = new ArrayBuffer(5);
var view = new Int32Array(buf, 0, 1);
view[0] = 256;
buf[4] = 5;
console.log(view[0]);
```
#### 创建一个空View对象

第二种构造函数用于创建一个预定义长度为 **n** 的空View对象,这个形式的构造函数还创建了一个足够大的`ArrayBuffer`对象来容纳 **n** 个View元素. 例如, [清单8-18](#listing-8-17) 创建了一个空的`Float32Array` View来容纳2个浮点数. 在这个构造函数的幕后, 还创建了一个长度为8字节的`ArrayBuffer`对象来容纳这两个浮点数. 在构造过程中, `ArrayBuffer`的所有字节被初始化为 **0**.

清单8-18. 创建一个空的 **Float32Array** View对象

```javascript` {#listing-8-18}
var view = new Float32Array(2);
```

#### 从数据值创建View对象

第三种形式的构造函数接受一个值数组用于填充View. 数组中的值被转换为核实的数据类型, 并存储在View中,此类构造函数还创建了一个全新的 `ArrayBuffer` 对象来容纳这些值.清单8-19显示了一个示例,用值 **1,2,3** 来填充一个`Uint16Array`

清单8-19. 从一个包含3个值的数组创建一个`Uint16Array`对象

```javascript` {#listing-8-19}
var view = new Uint16Array([1, 2, 3]);
```

#### 从其他View对象创建一个新的View对象

第四种构造函类似于第三种.唯一的不同是这类构造函数接受一个View作为唯一参数而不是一个数组. 新创建的View也初始化属于自己的后端ArrayBuffer, 其底层数据不和其他View共享. 清单8-20说明了这个版本的构造函数如如何使用. 在这个示例中, 一个4字节的`ArrayBuffer`用于创建一个`Int8Array` View来包含4个数字. `Int8Array` View然后用于创建一个新的`Uint32Array`View,`Uint32Array`也包含四个数字.但是其底层的`ArrayBuffer`是16个字节长度而不是4, 所以这两个View有不同的`ArrayBuffer`,修改其中一个不会影响另外一个.

清单 8-20. 从 **Int8Array** 创建一个 **Uint32Array**

```javascript` {#listing-8-20}
var buf = new ArrayBuffer(4);
var view1 = new Int8Array(buf);
var view2 = new Uint32Array(view1);
console.log(buf.byteLength); // 4
console.log(view1.byteLength); // 4
console.log(view2.byteLength); // 16
```

#### View属性

你已经知道, 一个View的底层`ArrayBuffer`对象可以通过其属性`buffer`访问, 其`BYTES_PER_ELEMENT`属性表示 **每个View元素** 的字节长度. View还有`byteLength`和`length`属性, 以及一个`byteOffset`属性表示View对象在底层`buffer`上的其实偏移量.

#### byteLength

`byteLength` 属性表示View的字节数据长度. 其值不需要等于底层`ArrayBuffer`的`byteLength`属性. 在[清单8-21](#listing-8-21)这个例子中, 一个`Int16Array`View是从一个10字节长的`ArrayBuffer`对象创建的. 因为`Int16Array`构造函数指明了其仅包含2个整数, 因此它的`byteLength`为4, 而`ArrayBuffer`的`byteLength`为10.

清单 8-21. **View** 对象的 **byteLength** 与其底层的 **ArrayBuffer** 对象的 **byteLength** 的区别

```javascript` {#listing-8-21}
var buf = new ArrayBuffer(10);
var view = new Int16Array(buf, 0, 2);
console.log(buf.byteLength);
console.log(view.byteLength);
```

#### length

`length`属性,类似于标准的数字, 表示在View中的元素数量. 如[清单8-22](#listing-8-22)所示, `length`属性通常用于遍历View对象中的元素.

清单 8-22. 使用 **length** 属性遍历 **View** 的元素

```javascript` {#listing-8-22}
var view = new Int32Array([5, 10]);
for (var i = 0, len = view.length; i < len; i++) {
    console.log(view[i]);
}
```

#### byteOffset

`byteOffset` 属性指明了View对象在底层的`ArrayBuffer`上的首字节位置. 如果不在构造函数中指定, 其值总是为0 [清单8-17](#listing-8-17). `byteOffset`可以和`byteLength`属性组合使用,遍历底层的`ArrayBuffer`. 在[清单8-23](#listing-8-23)这个例子中, 阐述了如何通过`byteOffset`和`byteLength`属性遍历只有View使用的字节,底层的`ArrayBuffer`为10字节长, 但View只使用了4-7字节.

清单 8-23. 遍历ArrayBuffer的子集

```javascript {#listing-8-23}
var buf = new ArrayBuffer(10);
var view = new Int16Array(buf, 4, 2);
var len = view.byteOffset + view.byteLength;
view[0] = 100;
view[1] = 256;
for (var i = view.byteOffset; i < len; i++) {
    console.log(buf[i]);
}
```

#### get()

`get()`方法用于获取View中给定索引的数据值, 也可以使用数组下标的方式获取,如[清单8-24](#listing-8-24)所示,使用`get()`来获取`View`对象中第一个元素`view[0]`的值.

清单 8-24. 使用`View`的`get()`方法

```javascript` {#listing-8-24}
var view = new Uint8ClampedArray([5]);
console.log(view.get(0));
// could also use view[0]
```

#### set()

`set()` 用作给一个或多的View中的元素赋值. 要给单个元素赋值,直接传递索引和值给`set()`方法. 也可以使用数组索引的方式赋值. 在[清单8-25](#listing-8-25)示例中, 值 **3.14** 被赋值给View的第四个元素.

清单 8-25. 使用`set()`给View赋值

```javascript` {#listing-8-25}
var view = new Float64Array(4);
view.set(3, 3.14);
// could also use view[3] = 3.14
```

`set()`还能接受一个数组或另一个View对象作为参数, 同时给多个元素赋值. 第二个可选参数为起始偏移量,用于指定从什么位置开始写. 如果不提供第二个参数, 默认从第一个索引位置开始写. 在[清单8-26](#listing-8-26)中, `set()`用于给`Int32Array`的所有元素赋值.

清单 8-26. Assigning Multiple Values Using set()

```javascript` {#listing-8-26}
var view = new Int32Array(4);
view.set([1, 2, 3, 4], 0);
```

[^2]这个版本的`set()`需要注意两点: 第一, 如果写操作超出了View的尾部, 将抛出异常. 在[清单8-26](#listing-8-26)中,如果第二个参数大于0, 那么就会超出View的边界, 将导致错误. 第二, 因为`set()`也可以接受一个View作为它的第一个参数,两个View可能共享底层相同的`ArrayBuffer`对象, 如果是这种情况, Node必须智能的复制数据,以使某些字节在拷贝之前不会被覆盖.在[清单8-27](#listing-8-27)中, 两个`Int8Array`View对象共享同样的`ArrayBuffer`对象, 第二个View `view2`为`view1`的一半,表示为`view1`的前半部分. 当调用`set()`的时候,`view2`的第一个元素值 **0** 被赋值给 **view1[1]**, `view2`的第二个元素值被赋值给**view1[2]**,在这个操作中,因为`view1[1]`是复制源`view2`(以及复制目标`view1`)的一部分, 你需要保证原来的值在被覆盖之前拷贝出来.



清单 8-27. 单个`ArrayBuffer`被多个View对象共享`set()`操作

```javascript` {#listing-8-27}
var buf = new ArrayBuffer(4);
var view1 = new Int8Array(buf);
var view2 = new Int8Array(buf, 0, 2);
view1[0] = 0;
view1[1] = 1;
view1[2] = 2;
view1[3] = 3;
view1.set(view2, 1);
console.log(view1.buffer);
```

~~According to the specification, "**setting the values takes place as if all the data is first copied into a temporary buffer that does not overlap either of the arrays, and then the data from the temporary buffer is copied into the current array**." Essentially, this means that Node takes care of everything for you. To verify this, the resulting output from the previous example is shown in Listing 8-28. Notice that bytes 1 and 2 hold the correct values of 0 and 1.~~

清单 8-28. 运行[清单8-27](#listing-8-27)的输出

```javascript` {#listing-8-28}
{ 
    '0': 0,
    '1': 0,
    '2': 1,
    '3': 3,
    slice: [Function: slice],
    byteLength: 4 
}
```

#### subarray()

`subarray()`, which returns a new view of the data type that relies on the same `ArrayBuffer`, takes two arguments.The first argument specifies the first index to be referenced in the new view. The second, which is optional, represents the last index to be referenced in the new view. If the ending index is omitted, the new view’s span goes from the start index to the end of the original view. Either index can be negative, meaning that the offset is computed from the end of the data array. Note that the new view returned by `subarray()` has the same `ArrayBuffer` as the original view. Listing 8-29 shows how `subarray()` is used to create several identical `Uint8ClampedArray` views making up a subset of another.

清单 8-29. 使用 `subarray()` 从现有的 View 创建一个新的Views 

```javascript`
var view1 = new Uint8ClampedArray([1, 2, 3, 4, 5]);
var view2 = view1.subarray(3, view1.length);
var view3 = view1.subarray(3);
var view4 = view1.subarray(-2);
```

## Node Buffers

Node provides its own `Buffer` data type for working with binary data. This is the preferred method of processing binary data in Node because it is slightly more efficient than typed arrays. Up to this point, you have encountered a number of methods that work with `Buffer` objects—for example, the fs module’s `read()` and `write()` methods. This section explores in detail how `Buffers` work, including their compatibility with the typed array specification.

### Buffer 构造函数

`Buffer` objects are created using one of the three `Buffer()` constructor functions. The `Buffer` constructor is global,meaning that it can be called without requiring any modules. Once a `Buffer` is created, it cannot be resized. The first form of the `Buffer`() constructor creates an empty `Buffer` of a given number of bytes. The example in Listing 8-30, which creates an empty 4-byte `Buffer`, also demonstrates that individual bytes within the `Buffer` can be accessed using array subscript notation.

清单 8-30. 创建一个4字节的Buffer并访问个别字节

```javascript` {#listing-8-30}
var buf = new Buffer(4);
buf[0] = 0;
buf[1] = 1;
console.log(buf);
```

Listing 8-31 shows the stringified version of the `Buffer`. The first two bytes in the `Buffer` hold the values 00 and 01,
which were individually assigned in the code. Notice that the final two bytes also have values, although they were never
assigned. These are actually the values already in memory when the program ran (if you run this code, the values you
see will likely differ), indicating that the `Buffer()` constructor does not initialize the memory it reserves to 0. This is
done intentionally—to save time when requesting a large amount of memory (recall that the `ArrayBuffer` constructor
initializes its buffer to 0). As `ArrayBuffer`s are commonly used in web browsers, leaving the memory uninitialized could
be a security hazard—you probably wouldn’t want arbitrary web sites to read the contents of your computer’s memory.
Since the `Buffer` type is specific to Node, it isn’t subject to the same security risks.

清单 8-31. The Output Resulting from Running the Code in Listing 8-30

```javascript
$ node buffer-constructor-1.js
<Buffer 00 01 05 02>
```

The second form of the `Buffer()` constructor accepts an array of bytes as its only argument. The resulting `Buffer` is populated with the values stored in the array. An example of this form of the constructor is shown in Listing 8-32.

清单 8-32. Creating a `Buffer` from an Array of Octets

```javascript`
var buf = new Buffer([1, 2, 3, 4]);
```

The final version of the constructor is used to create a `Buffer` from string data. The code in Listing 8-33 shows how a `Buffer` is created from the string "`foo`".

清单 8-33. Creating a Buffer from a String

```javascript
var buf = new Buffer("foo");
```

Earlier in this chapter, you learned that in order to convert from binary data to text, a character encoding must be specified. When a string is passed as the first argument to `Buffer()`, a second optional argument can be used to specify the encoding type. In Listing 8-33, no encoding is explicitly set, so UTF-8 is used by default. Table 8-2 breaks down the various character encodings supported by Node. (The astute reader might recognize this table from Chapter 5. However, it is worth repeating the information at this point in the book.) 

表格 8-2: The Various String Encoding Types Supported by Node 


Encoding Type   | Description
--------------  | -----------
utf8            | Multibyte encoded Unicode characters. UTF-8 encoding is used by many web pages and to
represent       | string data in Node.
ascii           | 7-bit American Standard Code for Information Interchange (ASCII) encoding.
utf16le         | Little-endian-encoded Unicode characters. Each character is 2 or 4 bytes.
ucs2            | This is simply an alias for utf16le encoding.
base64          | Base64 string encoding. Base64 is commonly used in URL encoding, e-mail, and similar applications.
binary          | 允许二进制数据只使用每个字符的前8位编码为字符串As this is deprecated in favor of the Buffer object, 将会在Node的后续版本删除.
hex             | Encodes each byte as two hexadecimal characters.

### Stringification Methods

Buffers can be stringified in two ways. The first uses the `toString()` method, which attempts to interpret the contents of the `Buffer` as string data. The `toString()` method accepts three arguments, all optional. They specify the character encoding and the starting and ending indexes of the Buffer to stringify. If unspecified, the entire Buffer is stringified using UTF-8 encoding. The example in Listing 8-34 stringifies an entire Buffer using `toString()`.

清单 8-34. Using the Buffer.toString() Method

```javascript
var buf = new Buffer("foo");
console.log(buf.toString());
```

The second stringification method, `toJSON()`, returns the Buffer data as a JSON array of bytes. You get a similar result by calling `JSON.stringify()` on the `Buffer` object. Listing 8-35 shows an example of the `toJSON()` method. 

清单 8-35. Using the Buffer.toJSON() Method

```javascript
var buf = new Buffer("foo");
console.log(buf.toJSON());
console.log(JSON.stringify(buf));
```

### Buffer.isEncoding()

The `isEncoding()` method is a class method (i.e., a specific instance is not needed to invoke it) that accepts a string as its only argument and returns a Boolean indicating whether the input is a valid encoding type. Listing 8-36 shows two examples of `isEncoding()`. The first tests the string "`utf8`" and displays true. The second, however, prints false because "`foo`" is not a valid character encoding. 

清单 8-36. Two Examples of the Buffer.isEncoding() Class Method

```javascript`
console.log(Buffer.isEncoding("utf8"));
console.log(Buffer.isEncoding("foo"));
```

### Buffer.isBuffer()

The class method `isBuffer()` is used to determine whether a piece of data is a `Buffer` object. It is used in the same fashion as the `Array.isArray()` method. Listing 8-37 shows an example use of `isBuffer()`. This example prints true because the buf variable is, in fact, a `Buffer`. 

清单 8-37. The Buffer.isBuffer() Class Method

```javascript`
var buf = new Buffer(1);
console.log(Buffer.isBuffer(buf));
```

### Buffer.byteLength() and length

The `byteLength()` class method is used to calculate the number of bytes in a given string. This method also accepts an optional second argument to specify the string’s encoding type. This method is useful for calculating byte lengths without actually instantiating a `Buffer` instance. However, if you have already constructed a `Buffer`, its length property serves the same purpose. In the example in Listing 8-38, which shows `byteLength()` and length, `byteLength()` is used to calculate the byte length of the string "`foo`" with UTF-8 encoding. Next, an actual `Buffer` is constructed from the same string. The `Buffer`’s length property is then used to inspect the byte length.

清单 8-38. Buffer.byteLength() and the length Property

```javascript
var byteLength = Buffer.byteLength("foo");
var length = (new Buffer("foo")).length;
console.log(byteLength);
console.log(length);
```


fill()
There are a number of ways to write data to a `Buffer`. The appropriate method can depend on several factors,including the type of data and its endianness. The simplest method, `fill()`, which writes the same value to all or part of a `Buffer`, takes three arguments—the value to write, an optional offset to start filling, and an optional offset to stop filling. As with the other writing methods, the starting offset defaults to 0, and the ending offset defaults to the end of the Buffer. Since a `Buffer` is not set to zero by default, `fill()` is useful for initializing a Buffer to a value. The example in Listing 8-39 shows how all the memory in a `Buffer` can be zeroed out.

清单 8-39. Zeroing Out the Memory in a Buffer Using fill()
```javascript`
var buf = new Buffer(1024);
buf.fill(0);
```

### write()

要写入一个字符串到`Buffer`,使用`write()`方法. 它接受下面四个参数:

- 要写入的字符串.
- 起始写入偏移量. 此参数为可选, 默认为索引0.
- pecified, 写入整个字符串, 如果`Buffer`的空间不够, 字符串将被截断.
- 字符串的编码, 默认为UTF-8.

The example in Listing 8-40 fills a 9-byte `Buffer` with three copies of the string "`foo`". As the first write starts at the beginning of the `Buffer`, an offset is not required. However, the second and third writes require an offset value. In the third, the string length is included though it is not necessary.

清单 8-40. 调用`write()`多次写入同一个`Buffer`

```javascript
var buf = new Buffer(9);
var data = "foo";
buf.write(data);
buf.write(data, 3);
buf.write(data, 6, data.length);
```

### Writing Numeric Data
There is a collection of methods used to write numeric data to a `Buffer`, each method being used to write a specific type of number. This is analogous to the various typed array views, each of which stores a different type of data.

表格 8-3: lists the methods used to write numbers.

方法名称         | 描述
-------------    | -----------
writeUInt8()     | 写入一个无符号8位整数.
writeInt8()      | 写入一个有符号8位整数.
writeUInt16LE()  | 使用小端字节顺序写入一个16位无符号整数.
writeUInt16BE()  | 使用大端字节顺序写入一个16位无符号整数.
writeInt16LE()   | 使用小端字节顺序写入一个16位有符号整数.
writeInt16BE()   | 使用大端字节顺序写入一个16位有符号整数.
writeUInt32LE()  | 使用小端字节顺序写入一个32位无符号整数.
writeUInt32BE()  | 使用大端字节顺序写入一个32位无符号整数.
writeInt32LE()   | 使用小端字节顺序写入一个32位有符号整数.
writeInt32BE()   | 使用大端字节顺序写入一个32位有符号整数.
writeFloatLE()   | 使用小端字节顺序写入一个32位浮点数.
writeFloatBE()   | 使用大端字节顺序写入一个32位浮点数.
writeDoubleLE()  | 使用小端字节顺序写入一个64位浮点数.
writeDoubleBE()  | 使用大端字节顺序写入一个64位浮点数.

All the methods in Table 8-3 take three arguments—the data to write, the offset in the `Buffer` to write the data, and an optional flag to turn off validation checking. If the validation flag is set to `false` (the default), an exception is thrown if the value is too large or the data overflows the `Buffer`. If this flag is set to true, large values are truncated, and overflow writes fail silently. In the example using `writeDoubleLE()` in Listing 8-41, the value 3.14 is written to the first 8 bytes of a `Buffer`, with no validation checking. 

清单 8-41. Using writeDoubleLE()

```javascript`
var buf = new Buffer(16);
buf.writeDoubleLE(3.14, 0, true);
```

### Reading Numeric Data

Reading numeric data from a Buffer, like writing, also requires a collection of methods. Table 8-4 lists various
methods used for reading data. Notice the one-to-one correspondence with the write methods in Table 8-3.


Table 8-4. The Collection of Methods Used for Reading Numeric Data from a Buffer

Method Name     | Description
--------------- | -------------
readUInt8()     | Reads an unsigned 8-bit integer.
readInt8()      | Reads a signed 8-bit integer.
readUInt16LE()  | Reads an unsigned 16-bit integer using little-endian format.
readUInt16BE()  | Reads an unsigned 16-bit integer using big-endian format.
readInt16LE()   | Reads a signed 16-bit integer using little-endian format.
readInt16BE()   | Reads a signed 16-bit integer using big-endian format.
readUInt32LE()  | Reads an unsigned 32-bit integer using little-endian format.
readUInt32BE()  | Reads an unsigned 32-bit integer using big-endian format.
readInt32LE()   | Reads a signed 32-bit integer using little-endian format.
readInt32BE()   | Reads a signed 32-bit integer using big-endian format.
readFloatLE()   | Reads a 32-bit floating-point number using little-endian format.
readFloatBE()   | Reads a 32-bit floating-point number using big-endian format.
readDoubleLE()  | Reads a 64-bit floating-point number using little-endian format.
readDoubleBE() | Reads a 64-bit floating-point number using big-endian format.

All the numeric read methods take two arguments. The first is the offset in the Buffer to read the data from. The optional second argument is used to disable validation checking. If it is `false` (the default), an exception is thrown if the offset exceeds the `Buffer` size. If the flag is true, no validation occurs, and the returned data might be invalid.Listing 8-42 shows how a 64-bit floating-point number is written to a buffer and then read back using `readDoubleLE()`.

清单 8-42. Writing and Reading Numeric Data

```javascript {#listing-8-42}
var buf = new Buffer(8);
var value;
buf.writeDoubleLE(3.14, 0);
value = buf.readDoubleLE(0);
```


### slice()

The slice() method returns a new `Buffer` that shares memory with the original `Buffer`. In other words, updates to the new `Buffer` affect the original, and vice versa. The `slice()` method takes two optional arguments, representing the starting and ending indexes to slice. The indexes can also be negative, meaning that they are relative to the end of the `Buffer`. Listing 8-43 shows how `slice()` is used to extract the first half of a 4-byte `Buffer`.

清单 8-43. 用 `slice()` 创建一个新的Buffer

```javascript {#listing-8-43}
var buf1 = new Buffer(4);
var buf2 = buf1.slice(0, 2);
```

### copy()

The `copy()` method is used to copy data from one `Buffer` to another. The first argument to `copy()` is the destination Buffer. The second, if present, represents the starting index in the target to copy. The third and fourth arguments, if present, are the starting and ending indexes in the source `Buffer` to copy. An example that copies the full contents of one `Buffer` to another is shown in Listing 8-44. 

清单 8-44. Copying the Contents of One Buffer to Another Using copy()

```javascript {#listing-8-44}
var buf1 = new Buffer([1, 2, 3, 4]);
var buf2 = new Buffer(4);
buf1.copy(buf2, 0, 0, buf1.length);
```


### Buffer.concat()

The `concat()` class method allows concatenation of multiple `Buffer`s into a single larger `Buffer`. The first argument to concat() is an array of `Buffer` objects to be concatenated. If no `Buffer`s are provided, `concat()` returns a zero-length `Buffer`. If a single `Buffer` is provided, a reference to that `Buffer` is returned. If multiple `Buffer`s are provided, a new `Buffer` is created. Listing 8-45 provides an example that concatenates two `Buffer` objects. 

清单 8-45. Concatenating Two Buffer Objects

```javascript {#listing-8-45}
var buf1 = new Buffer([1, 2]);
var buf2 = new Buffer([3, 4]);
var buf = Buffer.concat([buf1, buf2]);
console.log(buf);
```

### Typed Array Compatibility

`Buffer`s are compatible with typed array views. When a view is constructed from a `Buffer`, the contents of the `Buffer` are cloned into a new `ArrayBuffer`. The cloned `ArrayBuffer` does not share memory with the original `Buffer`. In the example in Listing 8-46, which creates a view from a `Buffer`, a 4-byte `Buffer` is cloned into a 16-byte `ArrayBuffer`,which backs a `Uint32Array` view. Notice that the `Buffer` is initialized to all 0s prior to creating the view. Without doing so, the view would contain arbitrary data.

清单 8-46. 从Buffer创建一个View

```javascript` {#listing-8-46}
var buf = new Buffer(4);
var view;
buf.fill(0);
view = new Uint32Array(buf);
console.log(buf);
console.log(view);
```

It is also worth pointing out that while a view can be constructed from a `Buffer`, `ArrayBuffers` cannot be.
A `Buffer` also cannot be constructed from an `ArrayBuffer`. A `Buffer` can be constructed from a view, but be cautious when doing so, as the views are likely to contain data that will not transfer well. In the simple example in Listing 8-47 illustrating this point, the integer 257, when moved from a Uint32Array view to a `Buffer`, becomes the byte value 1.

清单 8-47. Data Loss when Constructing a Buffer from a View

```javascript` {#listing-8-47}
var view = new Uint32Array([257]);
var buf = new Buffer(view);
console.log(buf);
```

## 结语

A lot of material has been covered in this chapter. Starting with an overview of binary data, you were exposed to topics including character encoding and endianness at a high level. From there, the chapter progressed into the typed array specification. Hopefully, you found this material useful. After all, it is part of the JavaScript language and can be used in the browser as well as in Node. After presenting `ArrayBuffers` and views, the chapter moved on to Node’s `Buffer` data type and, finally, looked at how the `Buffer` type works with typed arrays.

[^1]: Node中Buffer类型的slice()方法和JavaScript的`ArrayBuffer`类型的slice方法在语义上有很大的区别,Node中Buffer类型的slide()返回的是一个引用(类似指针),对其修改将会同时修改原始的Buffer对象, 而`ArrayBuffer`通过slice()方法调用创建的是一个全新的`ArrayBuffer`对象,此对象拥有独立的内存空间,使用的时候请特别注意这两者的区别.

[^2]: 译者注: 仔细阅读本段落,两个不同的`View`共享相同的底层`ArrayBuffer`的情况有些复杂
