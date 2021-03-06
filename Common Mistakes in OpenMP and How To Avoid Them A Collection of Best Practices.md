# OpenMP中的常见错误以及如何避免它们：最佳实践的集合 

**MichaelSüß和Claudia Leopold**

University of Kassel, Research Group Programming Languages / Methodologies, Wilhelmsh ̈oher Allee 73, D-34121 Kassel, Germany
 {msuess，leopold}@uni-kassel.de 

**摘要** 

关于使用OpenMP时常见错误的数据很少。本文介绍了在过去两年中我们的并行编程课程中观察到的编程错误，同时报告了有多少是编译器和工具能够自动发现的。对每种错误，本文都进行了解释，并为程序员今后如何避免它们提供了最佳实践。

## 1 引言 

OpenMP的主要设计目标之一是使并行编程更容易。然而，当新手程序员试图使用它时，仍然存在许多误解和陷阱。因此，我们对参加我们关于并行编程的讲座的85名学生进行了一项研究，并观察了他们在OpenMP作业中所犯的错误。该研究在第2节有详细描述。

在本文的剩余部分章节，我们将集中讨论最常见的错误，简要情况列在表1中，包括每年都有多少组（每组两名）学生犯错。我们将所有这些编程错误分为两类：

1. 正确性方面的错误：所有错误都会影响程序的正确性。 
2. 性能方面的错误：所有错误都会影响程序的速度，通常会导致程序变慢，但不会产生错误结果。

第3节更详细地解释了各类错误，同时，为新手程序员提出了可能的方法和最佳实践，以避免将来出现这些错误。第4节报告我们在各种OpenMP编译器上进行的测试，以确定利用编译器可以发现或纠正哪些编程错误。在第5节中，我们将所提出的所有建议列成一个OpenMP编程清单，同时给出我们自己编程经验中的其他建议。第6节综述了相关的工作。最后第7节给出小结。

| 问题| 2004年| 2005年| 总和|
| ---------------------------------------- | ---- | ---- | ---- |
| **影响正确性的错误** | | | |
| 1.访问不受保护的共享变量| 8 | 10 | 18 |
| 2.使用锁时缺少`flush` |7 | 11 | 18 |
| 3.在没有`flush`的情况下读取共享变量 | 5 | 10 | 15 |
| 4.忘记标记`private`变量 | 6 | 5 | 11 |
| 5.使用`ordered`子句而不使用`ordered`构造| 2 | 2 | 4 |
| 6.将`#pragma omp parallel for`中的循环变量声明为`shared` | 1 | 2 | 3 |
| 7.忘记在`#pragma omp parallel for`中的`for` | 2 | 0 | 2 |
| 8.在并行区开始后尝试更改线程数量 | 0 | 2 | 2 |
| 9.从非所有者线程中调用`omp_unset_lock（）` | 2 | 0 | 2 |
| 10.在`#pragma omp for`中尝试更改循环变量 | 0 | 2 | 2 |
| **影响性能的错误** | | | |
| 11.当`atomic`足够时，却使用了`critical` | 8 | 1 | 9 |
| 12.在`critical`区域内放置太多工作| 2 | 4 | 6 |
| 13.在并行区域外使用了孤立的构造 | 2 | 2 | 4 |
| 14.使用了不必要的`flush` | 3 | 1 | 4 |
| 15.使用了不必要的`critical` | 2 | 0 | 2 |
| **小组总数** | 26 | 17 | 43 |


表1.在OpenMP中编程时经常出错的列表 

## 2 调查方法 

本研究中，我们评估了两门课程，学生均为本科层次。第一门课程在2004-2005年冬季学期，第二门课程在2005-2006年冬季学期。第一门课程有51名学员（26组，大多数是每组两名学生），第二门课程有33名参加者（17组）。课堂内容包括对并行计算和并行算法的介绍，以及约5小时的OpenMP知识介绍。之后，要求学生以两人为一组完成编程作业，学生必须给老师（本文作者）当面讲解答辩。其后，我们分析了学生作业中常犯的错误类型，本文重点介绍与OpenMP有关的错误。

编程作业主要包括中小型的任务，其中包括： 

- 找出前N个素数。
- 使用多个线程模拟用餐哲学家的问题。
- 计算图的连接分量个数
- 为OpenMP指令/子句/函数编写测试用例 

总收集了231个程序，均使用了C/C++及OpenMP编程语言，并在各种编译器上进行测试（如来自SUN，Intel，Portland Group，IBM以及免费的OMPi编译器）。在开始讨论评估结果之前，这里值得给出说明的是：本文提供的编程错误当然跟我们讲授课程的方式有直接关系，对于我们课堂上讲授详细的那些主题，错误数量大大减少，而对于其它主题，学生们可依赖的只有OpenMP规范文档。此外，对于那些学生在提交作业前就已经纠正的错误，本文也不予考虑。为此，在使用表1中那些统计结果时请谨慎，它们仅仅给出一种新手程序员可能会犯错的可能性。

## 3 OpenMP中的常见错误以及如何避免这些错误的最佳实践 

本节将讨论我们观察到的最常犯的错误，并提出尽可能避免它们的解决方案。教师们有一个共识：如果课堂内容基于学生个人的作业和反馈，一旦指出，小组就很少再犯；但如果课堂上只以示例方式讲解则没有同样的效果。

有一些错误，我们目前也想不出避免它的最佳实践。以下先列举这些错误（编号跟表1相同）：

2. 使用锁时缺少`flush`：在OpenMP规范的2.5版之前，锁操作不包括`flush`。我们学生使用的编译器不符合OpenMP 2.5规范，因此我们必须将缺少的“flush”指令标记为编程错误。

5. 使用`ordered`子句而不使用`ordered`构造：这里的错误是将一个ordered子句直接放入for工作共享构造中，而不在for循环内的单独指定一个`ordered`子句。 

8. 在并行区开始后尝试更改线程数量：执行并行区域的线程数只能在区域开始之前更改。因此，尝试从并行区内更改此数字是错误的。

10. 在`#pragma omp for`中尝试更改循环变量：规范中明确禁止在循环内部更改循环变量。 

11. 当`atomic`足够时，却使用了`critical`：有一些特殊情况可以用简单的`atomic`结构实现同步。在这种情况下不使用它会导致程序可能更慢，因此是性能错误。

13. 在并行区域外使用了孤立的构造：当使用跟并行区组合的工作共享构造时，有时我们的学生会忘记在书写`parallel`，从而产生一个孤立的构造。还有一些情况下，完全忘记了并行区，导致了诸如孤立的`critical`构造。

14. 使用了不必要的`flush`：`flush`结构被编译器隐含地包含在代码的某些位置。在这些位置之前或之后再显式指定`flush`被认为是性能错误。

15. 使用了不必要的`critical`：这里的错误是使用`critical`结构来保护内存访问，尽管它们并不需要保护（例如对私有变量进行保护，或者对实际上只会有一个线程访问的内存仍然进行保护） 。 

### 3.1 访问不受保护的共享变量 

最常犯和最严重的错误是没有避免对同一个内存位置的并发访问。OpenMP提供了几种用于保护关键区域的构造，例如`critical`构造，`atomic`构造和锁。尽管在课堂讲座中介绍过所有这三种结构，但许多小组根本没有使用它们，或者会偶尔忘记使用它们。当被问到这个问题时，他们中的大多数都可以解释关键区域是什么以及如何使用这些构造，但如何代码中发现这些区域，对于新手并行程序员来说似乎很难。

让新手程序员意识到这个问题的一种方法是使用工具来诊断OpenMP程序。例如，Intel Thread Checker和Assure工具都可以找到对内存位置的并发访问。

### 3.2 读取没有“Flush”的共享变量

OpenMP内存模型是一个复杂的野兽。OpenMP规范中的所有部分都致力于它，甚至要花整篇论文讨论它[1]。这条错误就是其中复杂性的一个体现。简单地说，如果读取共享变量而不首先刷新它，那就不能保证它是最新的。实际上，问题更加复杂，因为不仅读取线程必须刷新该变量，而且预先写入的线程也需要刷新。许多学生没有意识到这一点，没有采取任何措施就开始直接读取共享变量。在许多常见架构中，因为经常进行自动刷新操作，故而这个问题并不会表现出来。此外，许多OpenMP结构中都包含隐式刷新，因而该问题在真实程序中也没有凸现出来。

还有一些情况下，学生通过将共享变量声明为`volatile`来避免这个问题，它在每次对该变量的读取之前和每次写入之后放置一个隐式的`flush`。当然，由于这样做还会禁用编译器的许多优化，因此通常是较差的解决方案。

正确的解决方案是让每个OpenMP程序员都意识到这个问题，明确说明对共享变量的每次读取都必须以`flush`开头，除非在这里没有讨论到的非常罕见的极端情况。这个`flush`可以由程序员明确写下来，也可以隐含在OpenMP构造中。

OpenMP规范的2.5版包含了关于内存模型的新段落。这是否足以使新手程序员意识到这个陷阱还有待观察。

### 3.3 忘记标记`private`变量 

在我们的研究中，这种编程错误经常出现。明明是以私有方式使用的变量，却忘记将其声明为私有变量，此时OpenMP的默认规则会将其视为共享变量。

对于C/C++程序员的第一个建议是，尽可能使用语言本身的提供的作用域规则。C和C++都允许在并行区域内声明变量，这些变量将是私有的（除了在规范中描述的罕见极端情况，例如静态变量），因此无需在OpenMP指令中明确声明，从而完全避免了这个错误。

对新手程序员的第二个建议是使用`default(none)`子句。这将要求在数据共享属性子句中显式声明每个变量，否则编译器会抱怨。我们不会建议将其作为OpenMP的默认行为，因为当前的默认规则对于有经验的程序员会更节约时间(不必在共享子句中放下每个共享变量）。

如果OpenMP编译器能提供一个开关，用于在并行区域的开头显示每个变量的数据共享属性，这也可能有所帮助。这将使程序员能够检查他们的所有变量是否都按预期进行了声明。这个功能可在那些检查OpenMP程序的第三方工具中加以实现。

该问题的另一个解决方案是使用Lin等人提出的自动扫描[2]。根据该提议，编译器负责确定所有数据的共享属性，会正确地将所涉及的变量私有化。Sun Compiler 9及更高版本中提供了这个功能。

最后值得一提的是，本错误导致本应私有的变量成为共享变量，前述已经提到的工具可以检测对共享变量的并发访问，因此这些工具应该发出警告并提醒程序员有问题。

### 3.4 在`#pragma omp parallel for`中声明循环变量为共享 

这个错误表明了对工作共享构造工作方式的明显误解。OpenMP规范明确指出这些变量被隐式转换为私有，并且我们测试的所有编译器都执行了转换。这里令人惊讶的事实是，许多编译器默默地进行转换，忽略了共享声明，甚至没有发出警告。来自编译器的更多警告消息肯定会有所帮助。

### 3.5 忘记在`#pragma omp parallel for`中的`for` 

这里的错误是，尝试使用组合的工作共享构造`#pragma omp parallel for`，但忘记书写最后的`for`。这将导致每个线程都完整地执行一遍整个循环，而不仅仅是程序员所期望的部分。

在大多数情况下，这个错误进一步又会产生第3.1节中的错误，因此可以使用那里指定的工具检测和避免。

完全避免该错误的一种方法是在使用`for`工作共享结构时显式指定所需的`schedule`子句。这对于可移植性来说是一个好主意，因为默认的`schedule`子句行为是跟具体的实现相关的。它还将导致编译器检测到这里的错误，因为规范不允许`#pragma omp parallel schedule（static）`这种格式，并会产生编译器错误。

### 3.6 从非所有者线程中调用`omp_unset_lock()` 

OpenMP规范明确指出： 

>设置锁的线程拥有锁。拥有锁的线程可以取消该锁定，将其恢复到解锁状态。一个线程不能设置(set)、或取消设置(unset)另一个线程所拥有的锁。（[3，p.102]）
>

我们的一些学生仍然犯了错误，试图从非所有者线程解锁。我们测试的大多数编译器甚至对此也不报错，但可能导致不可预期的结果。

为了避免这个错误，我们建议学生只在绝对必要时使用锁。大多数情况下，OpenMP提供的`critical`构造将足够且易于使用，仅在少数情况下使用锁（例如锁定可变大小数组的一部分）。

### 3.7 在`critical`区域内放置太多工作 

新手程序员容易出现这种编程错误，可能是由于他们对关键区域造成的代价缺乏足够的敏感性。该问题可分为两个子问题：

1. 在关键区域内放置远超其必要性的代码，可能阻塞住其他线程更长的时间。 

2. 经过关键区域的频次更多，超出其必要性，从而付出区域相关的更大维护成本。 

第一种情况的解决方案显而易见：程序员需要检查关键区域内的每一行代码是否真的需要存在。例如，大多数情况下，复杂的函数调用可以事先计算，没有必要出现在关键区域。

第二种情况，我们考虑以下的学生代码，用来查找数组中的最大值： 

```C
#pragma omp parallel for 
for (i = 0; i < N; ++i) { 
  #pragma omp critical 
  { 
    if (arr[i]>max)max=arr[i]; 
  } 
} 
```

由于`critical`区域出现在关键路径上，因此其开销发生了$ N $次。现在考虑以下这个略有改进的版本：

```C
#pragma omp parallel for 
for (i = 0; i < N; ++i) { 
  #pragma omp flush (max) 
  if (arr[i] > max) { 
    #pragma omp critical 
    { 
      max=arr[i]; 
    } 
  }
}
```

这个版本会更快（至少在架构上，“flush”操作明显快于关键区域），因为进入关键区域的频率较低。最后，考虑这个版本：

```C
#pragma omp parallel 
{ 
  int priv_max; 
  #pragma omp for 
  for (i = 0; i < N; ++i) {
    if (arr[i] > priv_max) priv max = arr[i]; 
  } 
  #pragma omp flush (max) 
  if (priv_max > max) { 
    #pragma omp critical 
    {
      max = priv_max; 
    } 
  }
}
```

这基本上是重新实现了使用`max`运算符的归约操作，之所以这样，是因为在OpenMP规范中，使用`max`运算符的归约仅针对Fortran语言（这本身就是引发学生迷惑混淆的原因）。以这种方式编写程序，并将其中的技巧展示给新手程序员，有助于他们更加深入理解性能问题。

## 4 编译器和工具 

有许多不同的编译器都支持OpenMP规范，我们想知道，哪些能够检测到第3节中描述的编程错误。为此，我们为每个错误编写了一个简短的测试用例。表2报告了使用不同编译器的测试结果。

表2. 怎样使用编译器诊断和发现问题 

| 档案                                 | icc  | pgcc | sun   | guide | xlc  | ompi | assure | itc  |
| ------------------------------------ | ---- | ---- | ------ | ----- | ---- | ---- | ------ | ---- |
| **影响正确性的错误**                 |      |      |        |       |      |      |        |      |
| 1. access_shared                     |      |      |        |       |      |      | eE     | eE   |
| 2. locks_flush                       |      |      |        |       |      |      |        |      |
| 3. read_shared_var                   |      |      |        |       |      |      |        | (eE) |
| 4. forget_private（= access_shared） |      |      |        |       |      |      | eE     | eE   |
| 5. ordered_without_ordered           |      |      |        |       |      |      |        | EW   |
| 6. shared_loop_var                   | cE   | cC   | cW + C | cE    | cC   | cC   | cE     | cE   |
| 7. forget_for                        |      |      |        |       |      |      | (eE)   | (eE) |
| 8. change_num_threads                |      |      |        |       |      |      |        |      |
| 9. unset_lock_diff_thread            |      |      |        |       |      |      |        |      |
| 10. change_loop_var                  |      |      |        |       | cW   |      |        |      |
| **影响性能的错误**                   |      |      |        |       |      |      |        |      |
| 11. crit_when_atomic                 |      |      |        |       |      |      |        |      |
| 12. too_much_crit（没有测试！）      |      |      |        |       |      |      |        |      |
| 13. orphaned_const                   |      |      | rW     |       |      |      |        |      |
| 14. unnec_flush                      |      |      |        |       |      |      |        |      |
| 15. unnec_crit                       |      |      |        |       |      |      |        |      |

第一列中，给出我们测试程序的名称，错误编号与表1相同。由于我们想不出如何测试问题12（在`critical`区域内放置太多工作），因此省略了该问题。测试程序4与测试程序1相同，因此结果也相同。表格的其余部分描述了以下编译器的结果：

* 英特尔编译器9.0（icc）

* Portland Group Compiler 6.0（pgcc）
* Sun编译器5.7（sun）
* KAP / Pro工具集C / C ++ 4.0的guide组件（guide）  
* IBM XL C / C ++企业版7.0（xlc）
* OMPi编译器0.8.2（ompi）
* KAP / Pro工具集C / C ++ 4.0的assure组件（assure）  
* 英特尔线程检查器2.2（itc） 

最后两个条目（assure和itc）本身不是编译器，而是帮助程序员在其OpenMP程序中发现错误的工具。据我们所知，sssure已经被英特尔线程检测器所超越并取代，但它仍然安装在许多计算中心。我们无法找到任何类似lint的工具来检查C语言中的OpenMP程序，但有一些商业上可用的Fortran解决方案。

表中使用的字母代码含义如下。

1. 第一个（非大写）字母表明编译时何时能检测出错误：c=编译时(compiletime)，r=运行时(runtime)，e=估值时(evalation)。其中，只有Assure和Intel Thread Checker会有估值步骤。
2. 第二个（大写）字母描述了编译器产生了何种响应：W=警告信息(warning)，E=错误信息(error)，C=自动转换(conversion)，其中，自动转换意思是说编译器自动修复了错误而没有生成警告。转换是针对问题6进行的，其中编译器将共享循环变量自动私有化。W + C表示编译器既生成了警告，但同时也修复了问题。
3. 最后一个约定是：当代码放在括号内时，表示程序发现了某个相关的问题，并且通过追溯可以找到真正的错误所在。例如：当程序员忘记在并行工作共享结构（问题7）中书写`for`时，它将导致数据竞争。英特尔线程检查器可以检测到这种竞争，从而使得原始错误导致的现象变得更加明显。

执行所有测试时，所有均使用编译器所支持的最高级别警告选项。

从表中可知，大多数编译器对于避免本文所述错误没有太大帮助。尽管诸如英特尔线程检查器之类的工具表现更好一些，但最重要的是还是程序员首先要避免错误。下一节中介绍的程序员清单是朝着这个方向迈出的一步。

## 5 OpenMP程序员清单 

本节总结了迄今为止给OpenMP新手程序员的建议，并将其重新命名为适合易于使用的检查清单表。这个清单是面向新手程序员的，其中有些条款是我们在使用OpenMP的经验的积累。

### 一般性错误认识 

* 使用OpenMP进行细粒度的并行性很有吸引力（在循环之前放置`pragma omp parallel for`）。不幸的是，由于线程创建和调度等开销，这很少会带来巨大的性能提升。因此，必须在更粗粒度并行性方面寻找潜力。

- 基于上一条款，对于嵌套循环，尽可能尝试并行化外层循环。循环重新排序技术有时可提供帮助。

- 适用时尽可能使用归约。如果OpenMP没有定义所需的归约运算，则可以像第3.7节那样自己自己实现它。

- 注意到目前仍有许多编译器不支持嵌套并行，或者，即使支持，也只是保证了正确性，并没有任何性能收益。

- 在进行I / O（针对屏幕或文件）时，首先将信息写入缓冲区（这个过程甚至可以并行完成），然后在一次推送到设备。 

- 使用多个编译器测试您的程序，并打开所有警告， 以便让不同的编译器发现不同的错误。 

- 使用英特尔线程检查器或Assure等工具检测编程错误，并指导您编写性能更好的程序。 

###并行区 

- 如果要为并行区指定执行的线程数，则调用`omp_set_num_threads()`（或其它指定线程数的方法 ）必须在进入并行区*之前*进行。 

- 如果程序依赖于并行区域中的线程数（例如，手动分配工作任务），请确保在进入并行区后再查询线程数量omp_get_num_threads()`。即便是禁止了线程动态调整功能，但运行时系统有时分配的线程数量也会少于预期！

- 尽力消除`private`子句，而是改在并行区内部声明私有变量。除了其他好处，这样做还会使数据共享属性子句更易于管理。

- 尽可能使用`default(none)`子句，它会让你更为关注变量的数据共享属性子句，并避免一些错误。 

###工作共享构造 

- 对于要并行化的每个循环，检查循环的每次迭代是否必须执行相同的工作量。如果工作量差别显著，则静态任务调度（通常是编译器中的默认调度方式）可能会损害性能，此时应该考虑`dynamic`或`guided`的任务调度。
- 在工作共享构造中明确指定调度类型，默认的调度方式会随具体实现而不同！ 

- 如果使用`ordered`，请记住总是必须同时使用`ordered`子句和`ordered`结构。 

###同步 

- 如果有多个线程访问变量，并且其中一个访问是写操作，则必须使用同步，即使它只是一个简单的操作（如$i=1$）也要如此，否则OpenMP无法保证任何结果！

- 只要可能，使用`atomic`而不是`critical`，因为编译器更容易优化`atomic`，而非`critical`。 

- 尽量在关键区域内放置尽可能少的代码。例如，复杂的函数调用通常可以预先执行。

- 尽量避免因为多次重复调用关键区而带来的额外成本，例如在进入关键区域之前先检查条件。 

- 只在必要时使用锁，其余情况尽可能使用`critical`子句。如果必须使用锁，请确保从同一个线程调用`omp_set_lock()`和`omp_unset_lock()`。

- 避免嵌套关键区域，如果真有需要，请特别注意避免死锁。 

- `critical`区通常是开销最大的同步构造（例如在许多机器架构上，由于要执行栅障操作，因而需要大约两倍的时间来执行），因此要对此进行针对性优化。但请记住，这里仅考虑的是实际执行同步本身的操作，而没有考虑线程必须在栅障上或关键区域之前等待的时间（这取决于程序结构、调度程序等多种因素）。 

###内存模型 

* 时刻牢记OpenMP的内存模型。即使您只读取共享变量，也必须事先将其刷新(flush)，除非出现了OpenMP规范中描述的非常罕见的极端情况。
* 在OpenMP 2.5之前，锁定操作并不意味着进行隐式刷新。 

## 6 相关工作 

我们不知道有关OpenMP中经常出错的任何其他研究。当然，在教科书[4]和教授OpenMP的演示文稿中，包含了一些错误警告以及提高性能的技术，但大多数时候，这些更多是关于并行编程的一般陷阱（例如避免死锁的警告）。有一个有趣的资源值得一提：袁林的博客[5]，他已经开始用OpenMP来描述经常犯的错误。有趣的是，在撰写本文时，他没有触及我们所描述的任何错误，这使我们认为OpenMP规范中隐藏了更多错误来源，本文提供的OpenMP检查清单绝非全面。

## 7 小结 

在本文中，我们提出了一项关于OpenMP经常出错的研究。作者对两个学期参与并行编程课程的学生作业进行整理，列出他们最常见的错误来源。我们归纳了15个常见错误，提出了相应的最佳实践建议，以避免将来出现这些错误。这些最佳实践已被列入新手程序员的检查清单，并加入作者自己实践 的一些经验。事实表明，当前主流OpenMP编译器并不能保护程序员免于犯这些错误。

##致谢 

我们感谢Bj̈orn Knafla校对论文及其富有洞察力的评论。我们感谢在亚琛工业大学计算机中心，达姆施塔特技术大学和卡塞尔大学提供用于测试不同的编译器和硬件。最后，感谢我们课程的学生，没有他们，我们就不可能从初学者的角度来看待OpenMP。

##参考文献 

1. Hoeflinger, J.P., de Supinski, B.R.: The OpenMP memory model. In: Proceedings of the First International Workshop on OpenMP - IWOMP 2005. (2005) 
2. Lin, Y., Copty, N., Terboven, C., an Mey, D.: Automatic scoping of variables in par- allel regions of an OpenMP program. In: Proceedings of the Workshop on OpenMP Applications & Tools - WOMPAT 2004. (2004) 
3. OpenMP Architecture Review Board: OpenMP specifications. http://www.openmp. org/specs (2005) 
4. Chandra, R., Dagum, L., Kohr, D.: Parallel Programming in OpenMP. Morgan Kaufmann Publishers (2000) 
5. Lin, Y.: Reducing the complexity of parallel programming. http://blogs.sun. com/roller/page/yuanlin (2005) 

