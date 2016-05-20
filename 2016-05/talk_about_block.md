#Talk about block

看过很多关于block的文章，也看过block的源码，也用过`clang -rewrite-objc`命令来生成c格式的block实现，因为这个实现里没有加入ARC，
所以我看了一下汇编的结果，这里做一个总结，也许除了我之外的人都看不懂吧。

###前言

block被看做是一个对象，因为他有isa指针，根据isa指针指向的类对象，block被分成3种类型，分别是全局block、栈上的block和堆上的block。全局block只引用了全局变量，它自己也被存储在全局，copy它完全没有什么作用。一般我们用的block一开始都是栈上的block，栈上的block被保存在栈上，这里不想过多的讨论栈了，总之函数返回之后栈上的东西都无效了，所以要想能够一直使用block，那么就要把它转移到堆上，这个过程通过向它发送copy方法实现，他的过程就是在堆上malloc一块内存，memcopy过去，isa指针也会指向堆block的class对象。

####补充说明

这里补充说明一下，如果把一个block对象赋值给类成员变量，也会被发送copy。在栈转堆的过程中，如果被capture的变量是对象，那会被retain，注意：被retain只是在栈转堆的时候才发生的，而不是这个block被定义的时候或者block被执行的时候。

把block的数据从栈迁移到堆的过程当然不会只是memcopy这么简单，block还可能capture了变量，接下来要做的操作跟capture的变量类型有很大关系。

###Capture变量的类型

####1. 变量为scalar，以`int aInt`为例

Stack->Heap前
在block的struct内增加一个int类型的值，在block被定义的时候将aInt赋值给struct内的int。

Stack->Heap时
malloc一个block的内存，直接把栈上的block memcopy过去，over。

####2. 变量为对象，以`NSObject *obj`为例

Stack->Heap前
在block的struct内增加一个NSObject *的变量，在block被定义的时候将obj赋值给struct内的obj。

Stack->Heap时
因为capture的是一个对象，在转移的时候要retain这个object，所以在block的struct内，会有两个帮助函数（就叫它们object_assign, object_dispose），一个是在block被转移到堆上的时候调用，
一个在block被释放时调用。
这个过程也是，malloc一个block的内存，直接把栈上的block memcopy过去，然后调用object_assign，函数内部会把obj retain。

####3. 变量是__block scalar，以`__block int mutableInt`为例

Stack->Heap前
__block允许我们修改被capture的值，此时我们面临几个问题，首先是被capture的值是在栈上，block的作用域不止在栈上，如果出了栈，再修改这个值的时候肯定不可以修改原始的`mutableInt`的地址的值了，所以在转移时一定要把mutableInt也转移到堆上。第二个问题，因为在这个函数中，其他地方也可能修改`mutableInt`的值，那这个值在block也要被更改。这里通过增加了一层间接引用来实现。
第三个问题，可能有第二个block也capture了`mutableInt`，还要保证第二个block对mutableInt的操作反映在其他block中。困难重重。
苹果是这样做的，她为这个mutableInt实现了一个struct，结构大概是：
```
   struct {
     Ref *forward;
     int flags;
     int mutableInt;
   } Ref
```
首先把栈上的mutableInt使用Ref stack_ref替换，stack_ref.forward = &stack_ref，这个作用域中随后对mutableInt的引用全部使用
stack_ref.forward->mutableInt来引用。而block中capture的是Ref *pRef，指向了栈上的Ref，即pRef = &stack_ref。block中对mutableInt的
引用也使用pRef->forward->mutableInt来引用。

Stack->Heap时
malloc一个block的内存，直接把栈上的block memcopy过去，block中还有两个帮助函数，类似capture对象的情况，不过此时做的操作不同，不是
retain对象，而是malloc一块Ref的内存，生成heap_ref，将heap_ref->mutableInt = pRef->forward->mutableInt; heap_ref->forward = &heap_ref;
使堆上的pRef->forward即栈上的Ref指向heap_ref，pRef->forward->forward = pRef;再堆上的赋值给block上的pRef，pRef = heap_ref。
好了，这样就搞定了之前讨论的第一和第二个问题，对于第三个问题，假设这个Ref已经被拷贝到堆上了，此时第二个capture mutableInt的block也
要被转移到堆上了，进行刚刚讨论的过程，注意在调用帮助函数的时候，不是直接就malloc一块Ref的内存，如果这个stack_ref.forward已经在堆上，
他的flag会指明这一点，此时，仅仅增加head_ref的引用计数。搞定，实在是一个复杂的过程。

####4. 变量是__block 对象，以`__block NSObject *obj`为例

这里只与第三种情况做一个对比，相对于__block scalar，有改变的是在Ref中，增加了两个帮助函数，用在Ref被转移到堆上的时候被调用，就像
你想的那样，这里就对obj做了retain操作。

####5. 未完待续

关于`__weak NSObject *obj`，`clang -rewrite-objc`总是报错，反正就是在Stack->Heap时，帮助函数将block中的obj标记成__weak之类的吧。

###block变成堆block之后

一个block被转移到堆上之后，以后再向它发送copy方法，仅仅增加它的引用计数。
