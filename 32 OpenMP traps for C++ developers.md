# 32 OpenMP traps for C++ developers

**By Andrey Karpov, published on October 28, 2008, updated January 1, 2015**

May 2008

Abstract
Introduction
Logical errors
1. Missing /openmp
2. Missing parallel
3. Missing omp
4. Missing for
5. Unnecessary parallelization
6. Incorrect usage of ordered
7. Redefining the number of threads in a parallel section
8. Using a lock variable without initializing the variable
9. Unsetting a lock from another thread
10. Using lock as a barrier
11. Threads number dependency
12. Incorrect usage of dynamic threads creation
13. Concurrent usage of a shared resource
14. Shared memory access unprotected
15. Using flush with a reference type
16. Missing flush
17. Missing synchronization
18. An external variable is specified as threadprivate not in all units
19. Uninitialized local variables
20. Forgotten threadprivate directive
21. Forgotten private clause
22. Incorrect worksharing with private variables
23. Careless usage of the lastprivate clause
24. Unexpected values of threadprivate variables in the beginning of parallel sections
25. Some restrictions of private variables
26. Private variables are not marked as such
27. Parallel array processing without iteration ordering Performance errors
1. Unnecessary flush
2. Using critical sections or locks instead of the atomic directive
3. Unnecessary concurrent memory writing protection
4. Too much work in a critical section
5. Too many entries to critical sections

Conclusion
References

## 

 

## Abstract

Since multi-core systems are spreading fast, the problem of parallel programming becomes more and more urgent. However, even the majority of experienced developers are new to this sphere. The existing compilers and code analyzers allow finding some bugs, which appear during parallel code development. However, many errors are not diagnosed. The article contains description of a number of errors, which lead to incorrect behavior of parallel programs created with OpenMP.

 

## Introduction

Parallel programming appeared a long time ago. The first multiprocessor computer was created in 1960s. However, processors' performance increase has been achieved through clock frequency increment and multiprocessor systems have been rare until recently. The clock frequency increment slows down nowadays and processors' performance increase is achieved through multiple cores. Multi-core processors are spread widely, therefore the problem of parallel programming becomes more and more urgent. Earlier it was enough to install a CPU with a higher clock frequency or larger cash memory to increase a program's performance. Nowadays this approach is useless and a developer will have to modify the program in order to increase the program's performance.

Since parallel programming begins gaining in popularity only now, the process of an existing application parallelization or a new parallel program creation may become very problematic even for experienced developers since this sphere is new for them. Currently existing compilers and code analyzers allow finding only some (very few) potential errors. All other errors remain unrecorded and may increase debug and testing time significantly. Besides that, almost all errors of this kind cannot be stably reproduced. The article concerns the C++ language, since C++ programs are usually demanded to work fast. Since Visual Studio 2005 & 2008 support the OpenMP 2.0 standard, we will concern the OpenMP technology. OpenMP allows you to parallelize your code with minimal efforts - all you need to do is to enable the /openmp compiler option and add the needed compiler directives describing how the program's execution flow should be parallelized to your code.

This article describes only some of the potential errors which are not diagnosed by compilers, static code analyzers and dynamic code analyzers. However, we hope that this paper will help you understand some peculiarities of parallel development and avoid multiple errors.

Also, please note that this paper contains research results, which will be used in the VivaMP static analyzer development ([http://www.viva64.com/vivamp.php](http://www.viva64.com/vivamp.php)). The static analyzer will be designed to find errors in parallel programs created with OpenMP. We are very interested in receiving feedback on this article and learning more patterns of parallel programming errors.

The errors described in this article are split into logical errors and performance errors similar to the approach used in one of the references [[1](http://www.viva64.com/go.php?url=100)]. Logical errors are errors, which cause unexpected results, i.e. incorrect program behavior. Performance errors are errors, which decrease a program's performance.

First of all, let us define some specific terms which will be used in this article:

Directives are OpenMP directives which define code parallelization means. All OpenMP directives have the appearance of #pragma omp ...

Clauses are auxiliary parts of OpenMP directives. Clauses define how a work is shared between threads, the number of threads, variables access mode, etc.

Parallel section is a code fragment to which the #pragma omp parallel directive is applied.

The article is for developers who are familiar to OpenMP and use the technology in their programs. If you are not familiar with OpenMP, we recommend that you take a look at the document [[2](http://www.viva64.com/go.php?url=101)]. A more detailed description of OpenMP directives, clauses, functions and environment variables can be found in the OpenMP 2.0 specification [[3](http://www.viva64.com/go.php?url=102)]. The specification is duplicated in the MSDN Library and this form of specification is more handy, then the one in the PDF format.

Now, let us describe the potential errors which are badly diagnosed by standard compilers or are not diagnosed at all.

## Logical errors

 

## 1. Missing /openmp

Let's start with the simplest error: OpenMP directives will be ignored if OpenMP support is not enabled in the compiler settings. The compiler will not report an error or even a warning, the code simply will not work the way the developer expects.

OpenMP support can be enabled in the "Configuration Properties | C/C++ | Language" section of the project properties dialog.

## 2. Missing parallel

OpenMP directives have rather complex format, therefore first of all we are considering the simplest errors caused by incorrect directive format. The listings below show incorrect and correct versions of the same code:

Incorrectly:



Correctly:

```C
1#pragma omp parallel for 
2... // your code
```

 

```C
1#pragma omp parallel
2{
3  #pragma omp for
4  ... //your code
5}
```

The first code fragment will be successfully compiled, and the #pragma omp for directive will be simply ignored by the compiler. Therefore, a single thread only will execute the loop, and it will be rather difficult for a developer to find this out. Besides the #pragma omp parallel for directive, the error may also occur with the #pragma omp parallel sections directive.

 

## 3. Missing omp

A problem similar to the previous one occurs if you omit the omp keyword in an OpenMP directive [[4](http://www.viva64.com/go.php?url=103)]. Let's take a look at the following simple example:

Incorrectly:

```C
1#pragma omp parallel num_threads(2)
2{
3   #pragma single
4   {
5     printf("men");
6   }
7}
```

Correctly:

```
1#pragma omp parallel num_threads(2)
2{
3   #pragma omp single
4   {
5     printf("men");
6   }
7}
```

The "me" string will be printed twice, not once. The compiler will report the "warning C4068: unknown pragma" warning. However, warnings can be disabled in the project's properties or simply ignored by a developer.

 

## 4. Missing for

The #pragma omp parallel directive may be applied to a single code line as well as to a code fragment. This fact may cause unexpected behavior of the for loop shown below:

```
1#pragma omp parallel num_threads(2)
2for (int i = 0; i < 10; i++)
3    myFunc();   
```

If the developer wanted to share the loop between two threads, he should have used the #pragma omp parallel for directive. In this case the loop would have been executed 10 times indeed. However, the code above will be executed once in every thread. As the result, the myFunc function will be called 20 times. The correct version of the code is provided below:

```
1#pragma omp parallel for num_threads(2)
2for (int i = 0; i < 10; i++)
3    myFunc();   
```

 

## 5. Unnecessary parallelization

Applying the #pragma omp parallel directive to a large code fragment may cause unexpected behavior in cases similar to the one below:

```
1#pragma omp parallel num_threads(2)
2{                               
3    ... // N code lines
4    #pragma omp parallel for
5    for (int i = 0; i < 10; i++)
6    {           
7        myFunc();
8    }
9}
```

In the code above a forgetful or an inexperienced developer who wanted to share the loop execution between two threads placed the parallel keyword inside a parallel section. The result of the code execution will be similar to the previous example: the myFunc function will be called 20 times, not 10. The correct version of the code should look like this:

```
1#pragma omp parallel num_threads(2)
2{                               
3    ... // N code lines
4    #pragma omp for
5    for (int i = 0; i < 10; i++)
6    {           
7        myFunc();
8    }
9}
```

 

## 6. Incorrect usage of ordered

The ordered directive may cause problems for developers who are new to OpenMP [[1](http://www.viva64.com/go.php?url=100)]. Let's consider the following sample:

Incorrectly:

```
1#pragma omp parallel for ordered
2for (int i = 0; i < 10; i++)
3{           
4    myFunc(i);
5}
```

Correctly:

```
1#pragma omp parallel for ordered
2for (int i = 0; i < 10; i++)
3{           
4    #pragma omp ordered
5    {
6        myFunc(i);
7    }
8}
```

In the first code fragment the ordered clause will be simply ignored, because its scope was not specified. The loop will still be executed in a random order (which may sometimes become ascending order if you're lucky).

 

## 7. Redefining the number of threads in a parallel section

Now, let us consider more complex errors, which may be caused by insufficient understanding of the OpenMP standard. According to the OpenMP 2.0 specification [[3](http://www.viva64.com/go.php?url=102)] the number of threads cannot be redefined inside a parallel section. Such an attempt will cause run-time errors and program termination of a C++ program. For example:

Incorrectly:

```
1#pragma omp parallel
2{                               
3    omp_set_num_threads(2);
4    #pragma omp for
5    for (int i = 0; i < 10; i++)
6    {           
7        myFunc();
8    }
9}
```

Correctly:

```
1#pragma omp parallel num_threads(2)
2{                               
3    #pragma omp for
4    for (int i = 0; i < 10; i++)
5    {           
6        myFunc();
7    }
8}
```

Correctly:

```
1omp_set_num_threads(2)
2#pragma omp parallel 
3{                               
4    #pragma omp for
5    for (int i = 0; i < 10; i++)
6    {           
7        myFunc();
8    }
9}
```

 

## 8. Using a lock variable without initializing the variable

According to the OpenMP 2.0 specification [[3](http://www.viva64.com/go.php?url=102)] all lock variables must be initialized via the omp_init_lock or omp_init_nest_lock function call (depending on the variable type). A lock variable can be used only after initialization. An attempt to use (set, unset, test) an uninitialized lock variable In a C++ program will cause a run-time error.

Incorrectly:

```
1omp_lock_t myLock;
2#pragma omp parallel num_threads(2)
3{                               
4    ...
5    omp_set_lock(&myLock);
6    ...
7}
```

Correctly:

```
1omp_lock_t myLock;
2omp_init_lock(&myLock);
3#pragma omp parallel num_threads(2)
4{                               
5    ...
6    omp_set_lock(&myLock);
7    ...
8}
```

## 9. Unsetting a lock from another thread

If a lock is set in a thread, an attempt to unset this lock in another thread will cause unpredictable behavior [[3](http://www.viva64.com/go.php?url=102)]. Let's consider the following example:

Incorrectly:

```
01omp_lock_t myLock;
02omp_init_lock(&myLock);
03#pragma omp parallel sections
04{                               
05    #pragma omp section
06    {
07        ...
08        omp_set_lock(&myLock);
09        ...
10    }
11    #pragma omp section
12    {
13        ...
14        omp_unset_lock(&myLock);
15        ...
16    }
17}
```

This code will cause a run-time error in a C++ program. Since lock set and unset operations are similar to entering and leaving a critical section, every thread, which uses locks should perform both operations. Here is a correct version of the code:

Correctly:

```
01omp_lock_t myLock;
02omp_init_lock(&myLock);
03#pragma omp parallel sections
04{                               
05    #pragma omp section
06    {
07        ...
08        omp_set_lock(&myLock);
09        ...
10        omp_unset_lock(&myLock);
11        ...
12    }
13    #pragma omp section
14    {
15        ...
16        omp_set_lock(&myLock);
17        ...
18        omp_unset_lock(&myLock);
19        ...
20    }
21}
```

 

## 10. Using lock as a barrier

The omp_set_lock function blocks execution of a thread until the lock variable becomes available, i.e. until the same thread calls the omp_unset_lock function. Therefore, as it has already been mentioned in the description of the previous error, each of the threads should call both functions. A developer with insufficient understanding of OpenMP may try to use the omp_set_lock function as a barrier, i.e. instead of the #pragma omp barrier directive (since the directive cannot be used inside a parallel section, to which the #pragma omp sections directive is applied). As the result the following code will be created:

Incorrectly:

```
01omp_lock_t myLock;
02omp_init_lock(&myLock);
03#pragma omp parallel sections
04{                               
05    #pragma omp section
06    {       
07        ...
08        omp_set_lock(&myLock);
09        ...
10    }
11    #pragma omp section
12    {
13        ...
14        omp_set_lock(&myLock);
15        omp_unset_lock(&myLock);
16        ...
17    }
18}
```

Sometimes the program will be executed successfully. Sometimes it will not. This depends on the thread which finishes its execution first. If the thread which blocks the lock variable without releasing it will be finished first, the program will work as expected. In all other cases the program will infinitely wait for the thread, which works with the lock variable incorrectly, to unset the variable. A similar problem will occur if the developer will place the omp_test_lock function call inside a loop (and that is the way the function is usually used). In this case the loop will make the program hang, because the lock will never be unset.

Since this error is similar to the previous one, the fixed version of the code will remain the same:

Correctly:

```
01omp_lock_t myLock;
02omp_init_lock(&myLock);
03#pragma omp parallel sections
04{                               
05    #pragma omp section
06    {
07        ...
08        omp_set_lock(&myLock);
09        ...
10        omp_unset_lock(&myLock);
11        ...
12    }
13    #pragma omp section
14    {
15        ...
16        omp_set_lock(&myLock);
17        ...
18        omp_unset_lock(&myLock);
19        ...
20    }
21}
```

## 11. Threads number dependency

The number of parallel threads created during a program execution is not a constant value in general case. The number is usually equal to the number of processors by default. However, a developer can specify the number of threads explicitly (for example, using the omp_set_num_threads function or the num_threads clause, which has higher priority than the function). The number of threads can also be specified via the OMP_NUM_THREADS environment variable, which has the lowest priority. Therefore, the number of threads, which currently execute a parallel section, is a very unreliable value. Besides, the value may vary from one machine to another. The behavior of your code should not depend on the number of threads, which execute the code, unless you are entirely sure that this is really necessary.

Let's consider an example from the article [[5](http://www.viva64.com/go.php?url=104)]. The following program should have printed all letters of the English alphabet according to the developer's plan.

Incorrectly:

```
01omp_set_num_threads(4);
02#pragma omp parallel 
03{
04    int LettersPerThread = 26 / omp_get_num_threads();
05    int ThisThreadNum = omp_get_thread_num();
06    int StartLetter = 'a' + ThisThreadNum * LettersPerThread;
07    int EndLetter = 'a' + ThisThreadNum * LettersPerThread + LettersPerThread;
08    for (int i=StartLetter; i<EndLetter; i++)
09        printf ("%c", i);
10}
```

However, only 24 of 26 letters will be printed. The cause of the problem is that 26 (the total number of letters) do not contain 4 (the number of threads). Therefore, the two letters remaining will not be printed. To fix the problem one can either significantly modify the code so that the code will not use the number of threads or share the work between a correct number of threads (e.g. 2 threads). Suppose the developer decided not to use the number of threads in his program and let the compiler share work between threads. In this case the fixed version of the code will be similar to the following one:

Correctly:

```
1omp_set_num_threads(4);
2#pragma omp parallel for
3for (int i = 'a'; i <= 'z'; i++)
4{
5    printf ("%c", i);
6}
```

All iterations of the loop will surely be executed. One can specify the way the iterations are shared between threads using the schedule clause. Now, the compiler will share work between the threads and he will never forget about the two "additional" iterations. In addition, the resulting code is significantly shorter and readable.

 

## 12. Incorrect usage of dynamic threads creation

The dynamic keyword may appear in two different contexts in OpenMP: in the schedule(dynamic) clause and in the OMP_DYNAMIC environment variable, which makes a little mess of this. It is important to understand the difference between the two cases. One should not think that the schedule(dynamic) clause can be used only if the OMP_DYNAMIC variable is equal to true. The two cases are not related at all, actually.

The schedule(dynamic) clause means that iterations of a loop are split into chunks, which are dynamically shared between threads. When a thread finishes execution of a chunk, the thread will start executing the following "portion". If we apply this clause to the previous example, each of the 4 threads will print 6 letters and then the thread, which will become free first, will print the last 2 letters.

The OMP_DYNAMIC variable sets, whether the compiler can define the number of threads dynamically. The cause of a possible problem with this variable is that the variable's priority is even higher than the one of the num_threads clause. Therefore, if the variable's value is equal to true, the setting overrides num_threads, omp_set_num_threads and OMP_NUM_THREADS. If a program's behavior depends on the number of threads, this may cause unexpected results. This is another argument for creating code, which does not depend on the number of threads.

As experience has shown, the value of the OMP_DYNAMIC environment variable is equal to false by default in Visual Studio 2008. However, there is no guarantee that this situation will remain unchanged in the future. The OpenMP specification [[3](http://www.viva64.com/go.php?url=102)] states that the variable's value is implementation-specific. Therefore, if the developer from the previous example chose an easier way and decided to use the number of threads in his calculations instead of modifying the code significantly, he should make sure that the number of threads would always be equal to the one he needs. Otherwise the code will not work correctly on a four-processor machine.

Correctly:

```
01if (omp_get_dynamic())
02  omp_set_dynamic(0);
03omp_set_num_threads(2);
04#pragma omp parallel private(i)
05{
06    int LettersPerThread = 26 / omp_get_num_threads();
07    int ThisThreadNum = omp_get_thread_num();
08    int StartLetter = 'a' + ThisThreadNum * LettersPerThread;
09    int EndLetter = 'a' + ThisThreadNum * LettersPerThread + LettersPerThread;
10    for (i=StartLetter; i<EndLetter; i++)
11        printf ("%c", i);
12}
```

 

## 13. Concurrent usage of a shared resource

If we modify the previous example's code so that the code prints at least two or more letters at a time (not one by one in a random order as it currently does), we will observe one more parallel programming problem, the problem of concurrent shared resource usage. In this case the resource is the application's console. Let's consider a slightly modified example from the article [[6](http://www.viva64.com/go.php?url=105)].

Incorrectly:

```
1#pragma omp parallel num_threads(2)
2{ 
3    printf("Hello Worldn");
4}
```

In spite of the developer's expectations, the program's output on a two-processor machine will be similar to the following two lines:

```
1HellHell oo WorWlodrl
2d
```

The behavior is caused by the fact that the string output operation is not atomic. Therefore, the two threads will print their characters simultaneously. The same problem will occur if you use the standard output thread (cout) or any other object accessible to the threads as a shared variable.

If it is necessary to perform an action, which changes a shared object's state, from two threads, one should make sure that the action is performed by a single thread at a time. One can use locks or critical sections to achieve this. The most preferable approach will be discussed further.

Correctly:

```
1#pragma omp parallel num_threads(2)
2{ 
3    #pragma omp critical
4    {
5        printf("Hello Worldn");
6    }
7}
```

 

## 14. Shared memory access unprotected

This error is described in the article [[1](http://www.viva64.com/go.php?url=100)]. The error is similar to the previous one: if several threads are modifying a variable's value concurrently, the result is unpredictable. However, the error is considered separately from the previous one, because in this case the solution will be slightly different. Since an operation on a variable can be atomic, it is more preferable to use the atomic directive in this case. This approach will provide better performance than critical sections. Detailed recommendations on shared memory protection will be provided further.

Incorrectly:

```
1int a = 0;
2#pragma omp parallel
3{ 
4    a++;
5}
```

Correctly:

```
1int a = 0;
2#pragma omp parallel
3{ 
4    #pragma omp atomic
5    a++;
6}
```

Another possible solution is to use the reduction clause. In this case every thread will get its own copy of the a variable, perform all the needed actions on this copy and then perform the specified operation to merge all the copies.

Correctly:

```
1int a = 0;
2#pragma omp parallel reduction(+:a)
3{ 
4    a++;
5}
6printf("a=%dn", a); 
```

The code above, being executed by two threads, will print the "a=2" string.

 

## 15. Using flush with a reference type

The flush directive makes all the threads refresh values of shared variables. For example, if a thread assigns 1 to a shared variable a, it does not guarantee that another thread reading the variable will get 1. Please note that the directive refreshes only the variables' values. If an application's code contains a shared reference pointing to an object, the flush directive will refresh only the value of the reference (a memory address), but not the object's state. In addition, the OpenMP specification [[3](http://www.viva64.com/go.php?url=102)] states explicitly that the flush directive's argument cannot be a reference.

Incorrectly:

```
01MyClass* mc = new MyClass();
02#pragma omp parallel sections
03{
04    #pragma omp section
05    {
06        #pragma omp flush(mc)
07        mc->myFunc();
08        #pragma omp flush(mc)
09    }
10    #pragma omp section
11    {
12        #pragma omp flush(mc)
13        mc->myFunc();
14        #pragma omp flush(mc)
15    }
16}
```

The code below actually contains two errors: concurrent access to a shared object, which has already been described above, and usage of the flush directive with a reference type. Therefore, if the myFunc method changes the object's state, the result of the code execution is unpredictable. To avoid the errors one should get rid of concurrent usage of the shared object. Please note that the flush directive is executed implicitly at entry to and at exit from critical sections (this fact will be discussed later).

Correctly:

```
01MyClass* mc = new MyClass();
02#pragma omp parallel sections
03{
04    #pragma omp section
05    {
06        #pragma omp critical
07        {
08            mc->myFunc();
09        }
10    }
11    #pragma omp section
12    {
13        #pragma omp critical
14        {
15            mc->myFunc();
16        }
17    }
18}
```

 

## 16. Missing flush

According to the OpenMP specification [[3](http://www.viva64.com/go.php?url=102)], the directive is implied in many cases. The full list of such cases will be provided further. A developer may count upon this fact and forget to place the directive in a place where it is really necessary. The flush directive is **not** implied in the following cases:

- At entry to for.
- At entry to or exit from master.
- At entry to sections.
- At entry to single.
- At exit from for, single or sections, if the nowait clause is applied to the directive. The clause removes implicit flush along with the implicit barrier.

Incorrectly:

```
1int a = 0;
2#pragma omp parallel num_threads(2)
3{
4    a++;
5    #pragma omp single
6    {
7        cout << a << endl;
8    }
9}
```

Correctly:

```
01int i = 0;
02#pragma omp parallel num_threads(2)
03{
04    a++;
05    #pragma omp single
06    {
07        #pragma omp flush(a)
08        cout << a << endl;
09    }
10}
```

The latest version of the code uses the flush directive, but it is not ideal too. This version lacks of synchronization.

 

## 17. Missing synchronization

Besides the necessity of the flush directive usage, a developer should also keep in mind threads synchronization.

The corrected version of the previous example does not guarantee that the "2" string will be printed to the application's console window. The thread executing the section will print the value of the a variable which was actual at the moment the output operation was performed. However, there is no guarantee that both threads will reach the single directive simultaneously. In general case the value might be equal to "1" as well as "2". This behavior is caused by missing threads synchronization. The single directive means that the corresponding section should be executed only by a single thread. However, it is equiprobable that the section will be executed by the thread which finishes its execution first. In this case the "1" string will be printed. A similar error is described in the article [[7](http://www.viva64.com/go.php?url=106)].

Implicit synchronization via an implied barrier directive is performed only at exit from the for, single or sections directive, if the nowait clause is not applied to the directive (the clause removes the implicit barrier). In all other cases the developer should take care of the synchronization.

Correctly:

```
01int i = 0;
02#pragma omp parallel num_threads(2)
03{
04    a++;
05    #pragma omp barrier 
06    #pragma omp single
07    {
08        cout<<a<<endl;
09    }
10}
```

This version of the code is entirely correct: the program will always print the "2" string. Please note that this version does not contain the flush directive since it is implicitly included in the barrier directive.

Now, let us consider one more example of missing synchronization. The example is taken from the MSDN Library [[8](http://www.viva64.com/go.php?url=107)].

Incorrectly:

```
01struct MyType 
02{
03    ~MyType();
04};
05MyType threaded_var;
06#pragma omp threadprivate(threaded_var)
07int main() 
08{
09    #pragma omp parallel
10    {
11      ...
12    }
13}
```

The code is incorrect, because there is no synchronization at exit from the parallel section. As the result, when the application's process execution finishes, some of the threads will still exist and they will not get a notification about the fact that the process execution is finished. The destructor of the threaded_var variable will actually be called only in the main thread. Since the variable is threadprivate, its copies created in other threads will not be destroyed and a memory leak will occur. It is necessary to implement synchronization manually in order to avoid this problem.

Correctly:

```
01struct MyType 
02{
03    ~MyType();
04};
05MyType threaded_var;
06#pragma omp threadprivate(threaded_var)
07int main() 
08{
09    #pragma omp parallel
10    {
11        ...
12        #pragma omp barrier
13    }    
14}
```

 

## 18. An external variable is specified as threadprivate not in all units

We're beginning to discuss the most troublesome errors: the errors related to the OpenMP memory model. And this one is the first error of this type. The concurrent access to shared memory can also be treated as an error related to the OpenMP memory model since the error is related to shared variables and all global-scope variables are shared by default in OpenMP.

Before we start discussing memory model errors, please note that they all are related to private, firstprivate, lastprivate and threadprivate variables. One can avoid most of the errors if he avoids using the threadprivate directive and the private clause. We recommend declaring the needed variables as local variables in parallel sections instead.

Now, when you're warned, let's start discussing the memory model errors. We'll start with the threadprivate directive. The directive is usually applied to global variables, including external variables declared in another units. In this case the directive should be applied to the variable in all the units in which the variable is used. This rule is described in the abovementioned MSDN Library article [[8](http://www.viva64.com/go.php?url=107)].

A special case of this rule is another rule described in the same article: the threadprivate directive cannot be applied to variables declared in a DLL which will be loaded via the LoadLibrary function or the /DELAYLOAD linker option (since the LoadLibrary function is used implicitly in this case).

 

## 19. Uninitialized local variables

When a thread starts, local copies of threadprivate, private and lastprivate variables are created for this thread. The copies are unitialized by default. Therefore, any attempt to work with the variables without initializing them, will cause a run-time error.

Incorrectly:

```
1int a = 0;
2#pragma omp parallel private(a)
3{
4    a++;
5}
```

Correctly:

```
1int a = 0;
2#pragma omp parallel private(a)
3{
4    a = 0;
5    a++;
6}
```

Please note that there is no need to use synchronization and the flush directive since every thread has its own copy of the variable.

 

## 20. Forgotten threadprivate directive

Since the threadprivate directive is applied only once and used for global variables declared in the beginning of a unit, it's easy to forget about the directive: for example, when it's necessary to modify a unit created half a year ago. As the result, the developer will expect a global variable to become shared, as it should be by default. However, the variable will become local for every parallel thread. According to the OpenMP specification [[3](http://www.viva64.com/go.php?url=102)], the variable's value after a parallel section is unpredictable in this case.

Incorrectly:

```
01int a;
02#pragma omp threadprivate(a)    
03int _tmain(int argc, _TCHAR* argv[])
04{
05    ...
06    a = 0;
07    #pragma omp parallel
08    {               
09        #pragma omp sections
10        {
11            #pragma omp section 
12            {
13                a += 3;
14            }
15            #pragma omp section
16            {
17                a += 3;
18            }
19        }
20        #pragma omp barrier
21    }
22    cout << "a = " << a << endl;
23}
```

The program will behave as described in the specification: sometimes "6" (the value the developer expects) will be printed in a console window. Sometimes, however, the program will print "0". This result is more logical, since 0 is the value assigned to the variable before the parallel section. In theory, the same behavior should be observed if the a variable is declared as private or firstprivate. In practice, however, we have reproduced the behavior only with the threadprivate directive. Therefore, the example above contains this directive. In addition, this case is the most probable.

This fact, however, does not mean that the behavior in the other two cases will be correct in all other implementations; so, one should consider the cases too.

Unfortunately, it is difficult to provide a good solution in this case, because removing the threadprivate directive will change the program's behavior and declaring a threadprivate variable as shared is forbidden by OpenMP syntax rules. The only possible workaround is to use another variable.

Correctly:

```
01int a;
02#pragma omp threadprivate(a)    
03int _tmain(int argc, _TCHAR* argv[])
04{
05    ...
06    a = 0;
07    int b = a;
08    #pragma omp parallel
09    {               
10        #pragma omp sections
11        {
12            #pragma omp section 
13            {
14                b += 3;
15            }
16            #pragma omp section
17            {
18                b += 3;
19            }
20        }
21        #pragma omp barrier
22    }
23    a = b;
24    cout << "a = " << a << endl;
25}
```

In this version the a variable becomes a shared variable for the parallel section. Of course, this solution is not the best one. However, this solution guarantees that the old code will not change its behavior.

We recommend that beginners use the default(none) clause to avoid such problems. The clause will make the developer specify access modes for all global variables used in a parallel section. Of course, this will make your code grow, but you will avoid many errors and the code will become more readable.

 

## 21. Forgotten private clause

Let's consider a scenario similar to the previous case: a developer needs to modify a unit created some time ago and the clause defining a variable's access mode is located far enough from the code fragment to be modified.

Incorrectly:

```
01int a;
02#pragma omp parallel private(a)
03{
04...
05a = 0;
06#pragma omp for
07for (int i = 0; i < 10; i++)
08{
09    a++;
10}
11#pragma omp critical
12{
13    cout << "a = " << a;
14}
15}
```

This error seems to be an equivalent of the previous one. However, it is not true. In the previous case the result was printed after a parallel section and in this case the value is printed from a parallel section. As the result, if the variable's value before the loop is equal to zero, the code will print "5" instead of "10" on a two-processor machine. The cause of the behavior is that the work is shared between two threads. Each thread will get its own local copy of the a variable and increase the variable five times instead of the expected ten times. Moreover, the resulting value will depend on the number of threads executing the parallel section. By the way, the error will also occur if one uses the firstprivate clause instead of the private clause.

Possible solutions are similar to the ones provided for the previous case: one should either significantly modify all older code or modify the new code so that it will be compatible with the behavior of the old code. In this case the second solution is more elegant than the one provided for the previous case.

Correctly:

```
01int a;
02#pragma omp parallel private(a)
03{
04...
05a = 0;
06#pragma omp parallel for
07for (int i = 0; i < 10; i++)
08{
09    a++;
10}
11#pragma omp critical
12{
13    cout << "a = " << a;
14}
15}
```

 

## 22. Incorrect worksharing with private variables

The error is similar to the previous one and opposite to the "Unnecessary parallelization" error. In this case, however, the error can be caused by another scenario.

Incorrectly:

```
01int a;
02#pragma omp parallel private(a)
03{
04    a = 0;
05    #pragma omp barrier
06    #pragma omp sections 
07    {   
08        #pragma omp section
09        {
10            a+=100;     
11        }       
12        #pragma omp section
13        {           
14            a+=1;
15        }
16    }   
17    #pragma omp critical
18{
19    cout << "a = " << a << endl;
20}
21}
```

In this case a developer wanted to increase the value of each local copy of the a variable by 101 and used the sections directive for this purpose. However, since the parallel keyword was not specified in the directive, no additional parallelization was made. The work was shared between the same threads. As the result, on a two-processor machine one thread will print "1" and the other one will print "100". If the number of threads is increased, the results will be even more unexpected. By the way, if the a variable is not declared as private, the code will become correct.

In the sample above it is necessary to perform additional code parallelization.

Correctly:

```
01int a;
02#pragma omp parallel private(a)
03{
04    a = 0;
05    #pragma omp barrier
06    #pragma omp parallel sections 
07    {   
08        #pragma omp section
09        {           
10            a+=100;     
11        }       
12        #pragma omp section
13        {           
14            a+=1;
15        }
16    }   
17    #pragma omp critical
18{
19    cout<<"a = "<<a<<endl;
20}
21}
```

 

## 23. Careless usage of the lastprivate clause

OpenMP specification states that the value of a lastprivate variable from the sequentially last iteration of the associated loop, or the lexically last section directive is assigned to the variable's original object. If no value is assigned to the lastprivate variable during the corresponding parallel section, the original variable has indeterminate value after the parallel section. Let's consider an example similar to the previous one.

Incorrectly:

```
01int a = 1;
02#pragma omp parallel 
03{
04    #pragma omp sections lastprivate(a)
05    {   
06        #pragma omp section 
07        {               
08            ...
09            a = 10;
10        }   
11        #pragma omp section 
12        {       
13            ...
14        }
15    }
16#pragma omp barrier
17}
```

This code may potentially cause an error. We were unable to reproduce this in practice; however, it does not mean that the error will never occur.

If a developer really needs to use the lastprivate clause, he should know exactly what value would be assigned to the variable after a parallel section. In general case an error may occur if an unexpected value is assigned to the variable. For example, the developer may expect that the variable will get a value from the thread that finishes its execution last, but the variable will get a value from a lexically last thread. To solve this problem the developer should simply swap the sections' code.

Correctly:

```
01int a = 1;
02#pragma omp parallel 
03{
04    #pragma omp sections lastprivate(a)
05    {   
06        #pragma omp section 
07        {               
08            ...         
09        }   
10        #pragma omp section 
11        {       
12            ...
13            a = 10;
14        }
15    }
16#pragma omp barrier
17}
```

 

## 24. Unexpected values of threadprivate variables in the beginning of parallel sections

This problem is described in the OpenMP specification [[3](http://www.viva64.com/go.php?url=102)]. If the value of a threadprivate variable is changed before a parallel section, the value of the variable in the beginning of the parallel section is indeterminate.

Unfortunately, the sample code provided in the specification, cannot be compiled in Visual Studio since the compiler does not support dynamic initialization of threadprivate variables. Therefore, we provide another, less complicated, example.

Incorrectly:

```
01int a = 5;
02#pragma omp threadprivate(a)
03int _tmain(int argc, _TCHAR* argv[])
04{
05...
06a = 10;
07#pragma omp parallel num_threads(2)
08{
09    #pragma omp critical
10    {
11        printf("nThread #%d: a = %d", omp_get_thread_num(),a);
12    }
13}
14getchar();
15return 0;
16}
```

After the program execution one of the threads will print "5" and the other will print "10". If the a variable initialization is removed, the first thread will print "0" and the second one will print "10". One can get rid of the unexpected behavior only by removing the second assignment. In this case both threads will print "5" (in case the initialization code is not removed). Of course, such modifications will change the code's behavior. We describe them only to show OpenMP behavior in the two cases.

The resume is simple: never rely upon on your compiler when you need to initialize a local variable. For private and lastprivate variables an attempt to use uninitialized variables will cause a run-time error, which has already been described above. The error is at least easy to localize. The threadprivate directive, as you can see, may lead to unexpected results without any errors or warnings. We strongly recommend that you do not use this directive. In this case your code will become much more readable and the code's behavior will be easier to predict.

Correctly:

```
01int a = 5;
02int _tmain(int argc, _TCHAR* argv[])
03{
04...
05a = 10;
06#pragma omp parallel num_threads(2)
07{
08    int a = 10;
09    #pragma omp barrier
10    #pragma omp critical
11    {
12        printf("nThread #%d: a = %d", omp_get_thread_num(),a);
13    }
14}
15getchar();
16return 0;
17}
```

 

## 25. Some restrictions of private variables

The OpenMP specification provides multiple restrictions concerning private variables. Some of the restrictions are automatically checked by the compiler. Here is the list of restrictions which are not checked by the compiler:

- A private variable must not have a reference type.
- If a lastprivate variable is an instance of a class, the class should have a copy constructor defined.
- A firstprivate variable must not have a reference type.
- If a firstprivate variable is an instance of a class, the class should have a copy constructor defined.
- A threadprivate variable must not have a reference type.

In fact, all the restrictions result into two general rules: 1) a private variable must not have a reference type 2) if the variable is an instance of a class, the class should have a copy constructor defined. The causes of the restrictions are obvious.

If a private variable has a reference type, each thread will get a copy of this reference. As the result, both threads will work with shared memory via the reference.

The restriction, concerning the copy constructor, is quite obvious too: if a class contains a field which has a reference type, it will be impossible to copy an instance of this class memberwise correctly. As the result, both threads will work with shared memory, just like in the previous case.

An example demonstrating the problems is too large and is unlikely necessary. One should only remember a single common rule. If it is necessary to create a local copy of an object, an array or a memory fragment addressed via a pointer, the pointer should remain a shared variable. Declaring the variable as private is meaningless. The referenced data should be either copied explicitly or (when you're dealing with objects) entrusted to the compiler which uses the copy constructor.

 

## 26. Private variables are not marked as such

The error is described in the article [[1](http://www.viva64.com/go.php?url=100)]. The cause of the problem is that a variable which is supposed to be private was not marked as such and is used as a shared variable since this access mode is applied to all variables by default.

We recommend that you use the default(none) clause, which has already been mentioned above, to diagnose the error.

As you can see, the error is rather abstract and it is difficult to provide an example. However, the article [[9](http://www.viva64.com/content/articles/parallel-programming/?lang=en&content=parallel-programming&f=Parallel_programs_analysis.html)] describes a situation in which the error occurs quite explicitly.

Incorrectly:

```
01int _tmain(int argc, _TCHAR* argv[])
02{
03 const size_t arraySize = 100000;
04 struct T {
05   int a;
06   size_t b;
07 };
08 T array[arraySize];
09 {
10   size_t i;
11   #pragma omp parallel sections num_threads(2)
12   {
13     #pragma omp section
14     {
15       for (i = 0; i != arraySize; ++i)
16         array[i].a = 1;
17     }
18     #pragma omp section
19     {
20       for (i = 0; i != arraySize; ++i)
21         array[i].b = 2;
22     }
23   }
24 }
25 size_t i;
26 for (i = 0; i != arraySize; ++i)
27 {
28   if (array[i].a != 1 || array[i].b != 2)
29   {
30     _tprintf(_T("OpenMP Error!n"));
31     break;
32   }
33 }
34 if (i == arraySize)
35   _tprintf(_T("OK!n"));
36    getchar();
37    return 0;
38}
```

The program's purpose is simple: an array of two-field structures is initialized from two threads. One thread assigns 1 to one of the fields and the other assigns 2 to the other field. After this operation the program checks whether the array was initialized successfully.

The cause of the error is that both threads use a shared loop variable. In some cases the program will print the "OpenMP Error!" string. In other cases an access violation will occur. And only in rare cases the "OK!" string will be printed. The problem can be easily solved by declaring the loop variable as local.

Correctly:

```
01...
02   #pragma omp parallel sections num_threads(2)
03   {
04     #pragma omp section
05     {
06       for (size_t i = 0; i != arraySize; ++i)
07         array[i].a = 1;
08     }
09     #pragma omp section
10     {
11       for (size_t i = 0; i != arraySize; ++i)
12         array[i].b = 2;
13     }
14   }
15 }
16...
```

The article [[1](http://www.viva64.com/go.php?url=100)] contains a similar example, concerning for loops (the example is considered as a separate error). The author states that loop variable of a for loop shared via the for OpenMP directive should be declared as local. The situation seems to be equal to the one described above at fist sight. However, it not true.

According to the OpenMP standard loop variables are converted to private implicitly in such cases, even if the variable is declared as shared. The compiler will report no warnings after performing this conversion. This is the case described in the article [[1](http://www.viva64.com/go.php?url=100)], and the conversion is performed in this case indeed. However, in our example the loop is shared between threads using the sections directive, not the for directive, and in this case the conversion is not performed.

The resume is quite obvious: loop variables must never be shared in parallel sections. Even if the loop is shared between threads via the for directive, you should not rely on implicit conversion in this case.

 

## 27. Parallel array processing without iteration ordering

Parallelized for loops execution was not ordered in all previous examples (except the one concerning the ordered directive syntax). The loops were not ordered because there was no need to do this. In some cases, however, the ordered directive is necessary. In particular, you need to use the directive if an iteration result depends on a previous iteration result (this situation is described in the article [[6](http://www.viva64.com/go.php?url=105)]). Let's consider an example.

Incorrectly:

```
1int* arr = new int[10];
2for(int i = 0; i < 10; i++)
3    arr[i] = i;
4#pragma omp parallel for
5for (int i = 1; i < 10; i++)
6    arr[i] = arr[i - 1];
7for(int i = 0; i < 10; i++)
8    printf("narr[%d] = %d", i, arr[i]);
```

In theory the program should have printed a sequence of zeros. However, on a two-processor machine the program will print a number of zeros along with a number of fives. This behavior is caused by the fact that iterations are usually split equally between the threads by default. The problem can be easily solved using the ordered directive.

Correctly:

```
01int* arr = new int[10];
02for(int i = 0; i < 10; i++)
03    arr[i] = i;
04#pragma omp parallel for ordered
05for (int i = 1; i < 10; i++)
06{
07    #pragma omp ordered
08    arr[i] = arr[i - 1];
09}
10for(int i = 0; i < 10; i++)
11    printf("narr[%d] = %d", i, arr[i]);
```

## Performance errors

## 1. Unnecessary flush

All errors considered above affected the analyzed programs' logic and were critical. Now, let us consider errors which only affect a program's performance without affecting the program's logic. The errors are described in the article [[1](http://www.viva64.com/go.php?url=100)].As we have already mentioned above, the flush directive is often implied. Therefore, explicit flush directive in these cases is unnecessary. Unnecessary flush directive, especially the one used without parameters (in this case all shared memory is synchronized), can significantly slow down a program's execution.Here are the cases in which the directive is implied and there is no need to use it:The barrier directive

- At entry to and at exit from critical
- At entry to and at exit from ordered
- At entry to and at exit from parallel
- At exit from for
- At exit from sections
- At exit from single
- At entry to and at exit from parallel for
- At entry to and at exit from parallel sections

 

## 2. Using critical sections or locks instead of the atomic directive

The atomic directive works faster than critical sections, since many atomic operations can be replaced with processor commands. Therefore, it is more preferable to apply this directive when you need to protect shared memory during elementary operations. According to the OpenMP specification, the directive can be applied to the following operations:x binop= exprx++++xx----xHere ? is a scalar variable, expr is a scalar statement which does not involve the ? variable, binop is +, *, -, /, &, ^, |, <<, or >> operator which was not overloaded. In all other cases the atomic directive cannot be used (this condition is checked by the compiler).

Here is a list of shared memory protection means sorted by performance in descending order: atomic, critical, omp_set_lock.

 

## 3. Unnecessary concurrent memory writing protection

Any protection slows down the program's execution and it does not matter whether you use atomic operations, critical sections or locks. Therefore, you should not use memory protection when it is not necessary.

A variable should not be protected from concurrent writing in the following cases:

- If a variable is local for a thread (also, if the variable is threadprivate, firstprivate, private or lastprivate).
- If the variable is accessed in a code fragment which is guaranteed to be executed by a single thread only (in a master or single section).

 

## 4. Too much work in a critical section

Critical sections always slow down a program's execution. Firstly, threads have to wait for each other because of critical sections, and this decreases the performance increase you gain using code parallelization. Secondly, entering and leaving a critical section takes some time.

Therefore, you should not use critical sections where it is not necessary. We do not recommend that you place complex functions' calls into critical sections. Also, we do not recommend putting code which does not work with shared variables, objects or resources in critical sections. It is rather difficult to give exact recommendations on how to avoid the error. A developer should decide whether a code fragment should be put into critical section in every particular case.

## 5. Too many entries to critical sections

As we have already mentioned in the previous error description, entering and leaving a critical section takes some time. Therefore, if the operations are performed too often, this may decrease a program's performance. We recommend that you decrease the number of entries to critical sections as much as possible. Let's consider a slightly modified example from the article [[1](http://www.viva64.com/go.php?url=100)].

Incorrectly:

```
1#pragma omp parallel for
2for ( i = 0 ; i < N; ++i ) 
3{ 
4    #pragma omp critical
5    {   
6        if (arr[i] > max) max = arr[i];
7    } 
8}
```

If the comparison is performed before the critical section, the critical section will not be entered during all iterations of the loop.

Correctly:

```
01#pragma omp parallel for
02for ( i = 0 ; i < N; ++i ) 
03{ 
04    #pragma omp flush(max)
05    if (arr[i] > max)
06    {
07        #pragma omp critical
08        {   
09            if (arr[i] > max) max = arr[i];
10        }
11    }
12}
```

Such a simple correction may allow you to increase your code's performance significantly and you should not disregard this advice.

## 

## Conclusion

This paper provides the most complete list of possible OpenMP errors, at least at the moment the paper was written. The data provided in this article were collected from various sources as long as from authors' practice. Please note that all the errors are not diagnosed by standard compilers.

All the errors can be divided into three general categories:

- Ignorance of the OpenMP syntax.
- Misunderstanding of the OpenMP principles.
- Incorrect memory processing (unprotected shared memory access, lack of synchronization, incorrect variables' access mode, etc.).

Of course, the errors list provided in this paper is not complete. There are many other errors which were not considered here. It is possible that more complete lists will be provided in new articles on this topic.

Most of the errors can be diagnosed automatically by a static analyzer. Some (only a few) of them can be detected by Intel Thread Checker. However, a specialized tool has not been created yet. In particular, Intel Thread Checker detects concurrent shared memory access, incorrect usage of the ordered directive and missing for keyword in the #pragma omp parallel for directive [[1](http://www.viva64.com/go.php?url=100)].

A program for visual representation of code parallelization and access modes could also be useful for developers and has not been created yet.

The authors start working on the VivaMP static analyzer at the moment. The analyzer will diagnose the errors listed above and, maybe, some other errors. The analyzer will significantly simplify errors detection in parallel programs (note that almost all such errors cannot be stably reproduced). Additional information on the VivaMP project can be found on the project page: ([http://www.viva64.com/vivamp.php](http://www.viva64.com/vivamp.php)).

 

## References

1. Michael Suess, Claudia Leopold, Common Mistakes in OpenMP and How To Avoid Them - A Collection of Best Practices, [http://www.viva64.com/go.php?url=100](http://www.viva64.com/go.php?url=100).
2. OpenMP Quick Reference Sheet, [http://www.viva64.com/go.php?url=101](http://www.viva64.com/go.php?url=101).
3. OpenMP C and C++ Application Program Interface specification, version 2.0, [http://www.viva64.com/go.php?url=102](http://www.viva64.com/go.php?url=102).
4. Yuan Lin, Common Mistakes in Using OpenMP 1: Incorrect Directive Format, [http://www.viva64.com/go.php?url=103](http://www.viva64.com/go.php?url=103).
5. Richard Gerber, Advanced OpenMP Programming, [http://www.viva64.com/go.php?url=104](http://www.viva64.com/go.php?url=104).
6. Kang Su Gatlin and Pete Isensee. Reap the Benefits of Multithreading without All the Work, [http://www.viva64.com/go.php?url=105](http://www.viva64.com/go.php?url=105).
7. Yuan Lin, Common Mistakes in Using OpenMP 5: Assuming Non-existing Synchronization Before Entering Worksharing Construct,

