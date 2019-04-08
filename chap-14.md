# 第14章 具有不连续数据发送的MPI程序设计

MPI除了可以发送或接收连续的数据之外，还可以处理不连续的数据。其基本方法有两种：一是允许用户自定义新的数据类型（又称派生数据类型）；二是数据的打包与解包，即在发送方将不连续的数据打包到连续的区域，然后发送出去，在接收方将打包后的连续数据解包到不连续的存储空间。
本章分别对这两种方法进行介绍。

## 14.1 派生数据类型

前面所介绍的通信，只牵涉含有相同数据类型的相邻缓冲区。这对两种用户限制太大：一种是经常想传送含有不同数据类型值的消息的用户(例如，一个整数计数值跟着一些实数)；另一种是经常发送非连续数据的用户(例如发送矩阵的一个子块)。一种解决的办法是在发送端把非连续的数据打包到一个连续的缓冲区，在接收端再解包。这样做的缺点在于在两端都需要额外的内存到内存拷贝操作，甚至当通信域系统具有收集分散数据功能的时候也是如此。

为了介绍派生数据类型，首先介绍一种通用的数据类型描述方法——类型图。使用类型图可以用比较精确和形式化的方法来描述各种各样的类型。类型图是一系列二元组的集合，两个数据类型是否相同取决于它们的类型图是否相同。
类型图的二元组为如下形式：

    <基类型，偏移>

则类型图可表示为

    类型图 := { <基类型0，偏移0>,  <基类型1，偏移1>, ..., <基类型n-1，偏移n-1> }

基类型指出了该类型图中包括哪些基本的数据类型，而偏移则指出该基类型在整个类型图中的起始位置。
基类型可以是预定义类型或派生类型，偏移可正可负，也没有递增或递减的顺序要求。
而一个类型图中包括的所有基类型的集合称为某类型的类型表，表示为

    类型表 := { 基类型0, ..., 基类型n-1 }

将类型图和一个数据缓冲区的基地址结合起来，就可以说明一个通信缓冲区内的数据分布情况。

预定义数据类型是通用数据类型的特例，比如MPI_INT是一个预先定义好了的数据类型句柄，其类型图为`{(int, 0)}`，有一个基类型入口项int和偏移0。其它的基本数据类型与此相似。
数据类型的跨度被定义为该数据类型的类型图中从第一个基类型到最后一个基类型间的距离。
即如果某一个类型的类型图为

    typemap = { (type0, disp 0), ..., (type n-1, disp n-1) }

则该类型图的下界定义为

    lb(typemap) := min {disp j}, 0 =< j <= n-1

该类型图的上界定义为

    ub(typemap) := max { disp j + sizeof(type j) }, 0 =< j <= n-1
    
该类型图的跨度定义为

    extent(typemap) := ub(typemap) - lb(typemap) + ε

由于不同的类型有不同的对齐位置的要求，ε就是能够使类型图的跨度满足该类型的类型表中的所有的类型都能达到下一个对齐要求所需要的最小非负整数值。

假设`type={(double,0),(char,8)}`(一个double型的值在偏移0，后面在偏移8处跟一个字符值)，进一步假设double型的值必须严格分配到地址为8的倍数的存储空间，则该数据类型的extent是16(从9循环到下一个8的倍数)。
一个由一个字符或面紧跟一个双精度值的数据类型，其extent也是16。

## 14.2 新数据类型的定义

### 14.2.1 连续复制的类型生成

最简单的数据类型生成器是MPI_TYPE_CONTIGUOUS，它得到的新类型是将一个已有的数据类型按顺序依次连续进行复制后的结果。

---

    MPI_TYPE_CONTIGUOUS(count, oldtype, newtype) 
        IN  count   复制个数(非负整数)
        IN  oldtype 旧数据类型(句柄)
        OUT newtype 新数据类型(句柄)

    int MPI_Type_contiguous(int count, MPI_Datatype oldtype, MPI_Datatype *newtype)

    MPI_TYPE_CONTIGUOUS(COUNT, OLDTYPE, NEWTYPE, IERROR)
    INTEGER COUNT, OLDTYPE, NEWTYPE, IERROR
---

用MPI_TYPE_CONTIGUOUS构造新的数据类型，设原来的数据类型oldtype的类型图为

    {(doubel,0), (char,8)}

其中类型的跨度为extent=16，对旧类型重复的次数count=3，则newtype返回的新类型的类型图为

    {(double,0), (char,8), (double,16), (char,24), (double,32), (char,40)}

### 14.2.2 向量数据类型的生成

MPI_TYPE_VECTOR是一个更通用的生成器，允许复制一个数据类型到含有相等大小块的空间。每个块通过连接相同数量的旧数据类型的拷贝来获得，块与块之间的空间是旧数据类型的extent的整倍数。

---

    MPI_TYPE_VECTOR(count, blocklength, stride, oldtype, newtype)
    IN count        块的数量(非负整数) 
    IN blocklength  每个块中所含元素个数(非负整数)
    IN stride       各块第一个元素之间相隔的元素个数(整数)
    IN oldtype      旧数据类型(句柄) 
    OUT newtype     新数据类型(句柄)

    int MPI_Type_vector(int count, int blocklength, int stride,
        MPI_Datatype oldtype, MPI_Datatype *newtype) 
    
    MPI_TYPE_VECTOR(COUNT, BLOCKLENGTH, STRIDE, OLDTYPE, NEWTYPE, IERROR)
    INTEGER COUNT, BLOCKLENGTH, STRIDE, OLDTYPE, NEWTYPE, IERROR

---

假设旧数据类型oldtype的类型图为`{(double,0),(char,8)}`, `extent=16`，则

    MPI_TYPE_VECTOR(2,3,4,oldtype,newtype)

调用生成的数据类型的类型图为

    {(double,0), (char,8), (double,16), (char,24), (double,32), (char,40), 
    (double,64), (char,72), (double,80), (char,88), (double,96), (char,104)}

即两个块，每个旧类型有三个拷贝，相邻块之间的步长stride为4个元素。

调用

    MPI_TYPE_VECTOR(3,1,-2,oldtype,newtype)
    
将生成以下的数据类型：

    {(double,0), (char,8), (double,-32), (char,-24), (double,-64), (char,-56)}

调用

    MPI_TYPE_CONTIGUOUS(count, oldtype, newtype)
    
等价于调用

    MPI_TYPE_VECTOR(count, 1, 1, oldtype, newtype )
    
或调用

    MPI_TYPE_VECTOR(1, count, n, oldtype, newtype)
    
其中n为升序。

函数MPI_TYPE_HVECTOR和MPI_TYPE_VECTOR基本相同，只是stride不再是元素个数，而是字节数。

### 14.2.3 索引数据类型的生成

MPI_TYPE_INDEXED允许复制一个旧数据类型到一个块序列中（每个块是旧数据类型的一个连接），每个块可以包含不同的拷贝数目和具有不同的偏移。所有的块偏移都是旧数据类型extent的整倍数。

### 14.2.4 结构数据类型的生成

MPI_TYPE_STRUCT是最通用的类型生成器，它能够在上面介绍的基础上进一步允许每个块包含不同数据类型的拷贝。

### 14.2.5 新类型递交和释放

新定义的数据类型在使用之前必须先递交给MPI系统。一个递交后的数据类型，可以作为一个基本类型，用在数据类型生成器中产生新的数据类型。预定义数据类型不需要递交就可以直接使用。

递交操作用于递交新定义的数据类型。
注意这里的参数是指向该类型的指针（或句柄），而不是定义该类型的缓冲区的内容。

MPI_TYPE_FREE调用将以前已递交的数据类型释放，并且将该数据类型指针或句柄设为MPI_DATATYPE_NULL。
由该派生类型定义的新派生类型不受当前派生类型释放的影响。

释放一个数据类型并不影响另一个根据这个被释放的数据类型定义的其它数据类型。

## 14.3 地址函数 

MPI提供的地址调用MPI_ADDRESS，可以返回某一个变量在内存中相对于预定义的地址MPI_BOTTOM的偏移地址。

## 14.4 与数据类型有关的调用

MPI_TYPE_EXTENT以字节为单位返回一个数据类型的跨度extent。

MPI_TYPE_SIZE以字节为单位，返回给定数据类型有用部分所占空间的大小，即跨度减去类型中的空隙后的空间大小。
和MPI_TYPE_EXTENT相比，MPI_TYPE_SIZE不包括由于对齐等原因导致数据类型中的空隙所占的空间。

MPI_CET_COUNT返回的是以指定的数据类型为单位，接收操作接收到的数据的个数；而MPI_GET_ELEMENTS返回的则是以基本的类型为单位的数据的个数。
MPI_GET_COUNT和MPI_GET_ELEMENT对基本数据类型使用时返回值相同。

MPI_GET_COUNT从状态变量status中得到接收到的数据的个数，它是以指定的数据类型datatype为单位来计算的。

## 14.5 下界标记类型和上界标记类型

MPI提供两个特殊的数据类型，称为伪数据类型，即上界标记类型MPI_UB和下界标记类型MPI_LB。这两个数据类型不占空间即extent(MPI_LB)=extent(MPI_UB)=0。
它们主要是用来影响数据类型的跨度，从而对派生数据类型产生影响。

## 14.6 打包与解包

打包(Pack)和解包(Unpack)操作是为了发送不连续的数据，在发送前显式地把数据包装到一个连续的缓冲区，在接收之后从连续缓冲区中解包。

MPI_PACK把由inbuf, incount, datatype指定的发送缓冲区中的inbount个datatype类型的消息放到起始为outbuf的连续空间，该空间共有outcount个字节。
输入缓冲区可以是MPI_SEND允许的任何通信缓冲区。
入口参数position的值是输出缓冲区中用于打包的起始地址，打包后它的值根据打包消息的大小来增加；出口参数position的值是被打包的消息占用的输出缓冲区后面的第一个地址。
通过连续几次对不同位置的消息调用打包操作，就将不连续的消息放到了一个连续的空间。
comm参数是将在后面用于发送打包的消息时用的通信域。

