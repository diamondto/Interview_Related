# JS 深入话题  
## JavaScript 的数据是如何存储的  
一般来说，基本数据类型用栈存储，引用数据类型用堆存储  
但是，对于闭包变量来说，如果变量存在栈中，函数调用结束栈顶空间销毁，那闭包变量就没了  
> 闭包变量是存在堆内存的  
具体来说，以下数据类型存储在栈中：  
1. boolean  
2. null  
3. undefined  
4. number  
5. string  
6. symbol  
7. bigint  
而所有的对象数据类型存放在堆中  
- 为什么不全部用栈来保存？  
  对于系统栈来说，它的功能除了保存变量之外，还有创建并切换函数执行上下文的功能，会有一个入栈、出栈的操作过程  
  如果采用栈来存储相对基本类型更加复杂的对象数据，那么切换上下文的开销将会变得巨大  
  堆内存虽然空间大，能存放大量的数据，但同时垃圾内存的回收会带来更大的开销  

## V8 引擎如何进行垃圾内存的回收  
JS 类似 Java，采用一套自己的垃圾回收算法进行自动的内存管理  

- V8 内存限制  
  V8 只能使用系统的一部分内存，具体来说，在 64 位系统下，V8 最多只能分配 1.4G，在 32 位系统中，最多只能分配 0.7G  
  + V8 为什么要给堆内存设置上限？  
  1. JS 是单线程的执行机制  
     这意味着一旦进入到垃圾回收，那么其他的各种运行逻辑都要暂停  
  2. JS 垃圾回收其实是非常耗时的操作  
    > 以 1.5G 的垃圾回收堆内存为例，V8 做一次小的垃圾回收需要 50ms 以上，做一次非增量式的垃圾回收甚至要 1s 以上  
  1s 的时间对于前端来说是很长的一段时间，JS 代码执行会一直没有响应，造成应用卡顿，导致应用性能和响应能力直线下降，V8 作了一个简单粗暴的选择，就是选址堆内存  

- 所有调整堆内存的限制  
  当然也可以自己调整堆内存的限制  
  老生代部分内存调整(MB): `node --max-old-space-size=2048 xxx.js`  
  新生代部分内存调整(KB): `node --max-new-space-size=2048 xxx.js`  

- 新生代内存的回收  
  V8 把堆内存分成了两部分处理——新生代内存和老生代内存  
  1. 新生代内存：临时分配的内存，存活时间短  
  2. 老生代内存：常驻内存，存活的时间长  
  V8 的堆内存，也就是两个内存之和  
  **新生代垃圾回收过程** --- Scavenge 算法  
  1. 首先将新生代内存空间一分为二： From | To  
    其中 From 部分表示正在使用的内存，To 是目前闲置的内存  
    进行垃圾回收的时候，V8 将 From 部分的对象检查一遍，如果是存活对象就复制到 To 内存中(在 To 内存中按照顺序从头放置)，如果是非存活对象就直接回收  
  2. 当 From 中所有的存活对象顺序进入到 To 内存之后，将 From 和 To 两者的角色互换， From 成为新的闲置内存，To 成为正在新的使用的内存  
  **为什么不直接回收非存活对象**  
  为什么要将存活对象存入 To 内存中之后再交换角色？  
  To 内存中是按照顺序从头放置的，这是为了堆内存中存放的位置过散，导致闲置的空间比较零散，可能会导致占内存较大的对象无法进行空间分配  
  这样零散的空间叫做内存碎片  

- 老生代内存的回收  
  如果新生代中的变量经过多次回收后依然存在，那么就会被放入到老生代内存中，这种现象叫做晋升  
  会触发晋升的情况：  
  1. 已经经历过一次 Scavenge 回收  
  2. To(闲置)空间的内存占用超过 25%  
  **老生代垃圾回收过程**  
  老生代中累计的变量空间一般很大，所以不能使用 Scavenge 算法，采取标记清除的方式  
  1. 进行**标记-清除**，主要分为两个阶段：标记阶段和清除阶段。  
     首先会遍历堆中的所有对象，对它们做上标记，然后对代码环境中使用的变量以及被强引用的变量取消标记，剩下的标记变量就是要删除的变量  
     在随后的清除阶段对其进行空间的回收  
  2. 对于标记-清除也可能会造成内存碎片，V8 引擎对于老生代内存的选择是**整理内存碎片**  
     在清除阶段结束后，把存活的对象全部往一遍靠拢，由于是移动对象，它的执行速度不可能很快，**事实上也是整个过程中最耗时的部分**  

- 增量标记  
  由于 JS 单线程的机制，V8 在进行垃圾回收的时候，不可避免会产生阻塞业务逻辑的执行，如果老生代的垃圾回收任务很重，那么耗时会非常的可怕，严重影响性能，为了避免这样的问题，V8 采取了**增量标记**的方案  
  **增量标记：**将一口气完成的标记任务分为很多小的部分完成，每做完一个小的部分就'歇'一会儿，让 JS 逻辑执行一会儿，然后再执行下面的部分，如此循环直到标记阶段完成进入内存碎片的整理阶段  
  经过增量标记之后，垃圾回收过程对 JS 应用的阻塞时间减少到原来的 1/6  

## 描述一下 V8 执行一段 JS 代码的过程  
理解 V8 的执行机制，能够帮助理解很多的上层应用，包括 Babel、Eslint、前端框架的底层机制   

- 首先，机器是读不懂 JS 代码的，只能理解特定的机器码，如果要让 JS 的逻辑在机器上运行，就必须将 JS 的代码翻译成机器码，然后再让机器识别  
  JS 是解释型语言，对于解释型语言，解析器会对源代码做如下分析：  
  1. 通过词法分析和语法分析生成 AST(抽象语法树)  
  2. 生成字节码  
  然后根据字节码来执行程序  

### 1. 生成 AST(抽象语法树)  
  生成 AST 分为两步 —— 词法分析和语法分析  
  **词法分析就是分词**，它的工作就是将一行行的代码分解成一个个 token  
  `let name = 'Horace'`  
  上面的代码会被分成四个部分(token)  
  let name = 'Horace' --- 关键字 变量名 赋值 字符串  
  **语法分析阶段就是将生成的 token 数据，根据一定的语法规则转换为 AST**  
  生成 AST 之后，编译器/解释器后续的工作都要依靠 AST 而不是源代码  
  babel 的工作原理就是将 ES6 的代码解析生成 AST，然后将 ES6 的 AST 转换为 ES5 的 AST，最后才将 ES5 的 AST 转换为具体的 ES5 代码  

### 2. 生成字节码  
  生成 AST 之后，直接通过 V8 的解释器来生成字节码，**但是字节码并不能让机器直接运行**  
  V8 早期是直接把 AST 转换成机器码，但是机器码的体积太大，引发了严重的内存占用问题  
  字节码是比机器码轻量的多的代码  
  > 字节码是介于 AST 和机器码之间的一种代码，但是与特定类型的机器码无关，字节码需要通过解释器将其转换为机器码然后执行  
  字节码依然需要转换为机器码，但和原来不同的是，现在不需要一次性将全部的字节码都转换成机器码，而是通过解释器来逐步执行字节码，省去了二进制文件的操作，这样就大大降低了内存的压力  

### 3. 执行代码  
  接下来就是字节码解释执行的阶段  
  在执行字节码的过程中，如果发现某一部分代码重复出现，V8 将它记为 `热点代码(HotSpot)`，然后将这段代码编译成机器码保存起来，这个用来编译的工具就是 V8 的编译器，因此在这样的机制下，代码执行的时间越久，执行效率会越来越高，因为有越来越多的字节码被标记为 `热点代码`，遇到它们的时候直接执行相应的机器码，不用再次转换为机器码  
  
  字节码不仅配合了解释器，还和编译器打交道，所以 JS 并不是完全的解释型语言，而编译器和解释器的根本区别在于前者会编译生成二进制文件但后者不会  
  这种字节码跟编译器和解释器结合的技术叫做 `即时编译`，也就是 `JIT`  

### 总结  
  1. 首先通过词法分析和语法分析生成 AST  
  2. 将 AST 转换为字节码  
  3. 由解释器逐行执行字节码，遇到热点代码启动编译器进行编译，生成对应的机器码，以优化执行效率  

