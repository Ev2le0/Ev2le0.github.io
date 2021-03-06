[原文](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

By: David Howells <dhowells@redhat.com>

Paul E. McKenney <paulmck@linux.vnet.ibm.com>

翻译:sniperHW <huangweilook@21cn.com>

目录:

>1 内存访问抽象模型.
>
>- 操作外设.
>- 保证.

>2 什么是内存屏障?
>
>- 内存屏障的种类.
>- 对于内存屏障不能做什么假设?
>- 数据依赖屏障.
>- 控制依赖.
>- SMP屏障配对.
>- 内存屏障举例.
>- 读内存屏障VS内存预取.
>- 传递性.

>3 内核中的显式屏障.
>- 编译器屏障.
>- CPU内存屏障.
>- MMIO写屏障.

>4 内核中的隐式内存屏障.
>- Locking functions.
>- Interrupt disabling functions.
>- Sleep and wake-up functions.
>- Miscellaneous functions.

>5 Inter-CPU locking barrier effects.
>- Locks vs memory accesses.
>- Locks vs I/O accesses.

>6 什么时候需要内存屏障?
>- Interprocessor interaction.
>- Atomic operations.
>- Accessing devices.
>- Interrupts.

>7 Kernel I/O barrier effects.

>8 Assumed minimum execution ordering model.

>9 The effects of the cpu cache.
>- Cache coherency.
>- Cache coherency vs DMA.
>- Cache coherency vs MMIO.

>10 The things CPUs get up to.
>- And then there's the Alpha.

>11 Example uses.
>- Circular buffers.

>12 References.



#1 内存访问抽象模型

 考虑如下的抽象系统模型：

    		            :                :
    		            :                :
    		            :                :
    		+-------+   :   +--------+   :   +-------+
    		|       |   :   |        |   :   |       |
    		|       |   :   |        |   :   |       |
    		| CPU 1 |<----->| Memory |<----->| CPU 2 |
    		|       |   :   |        |   :   |       |
    		|       |   :   |        |   :   |       |
    		+-------+   :   +--------+   :   +-------+
    		    ^       :       ^        :       ^
    		    |       :       |        :       |
    		    |       :       |        :       |
    		    |       :       v        :       |
    		    |       :   +--------+   :       |
    		    |       :   |        |   :       |
    		    |       :   |        |   :       |
    		    +---------->| Device |<----------+
    		            :   |        |   :
    		            :   |        |   :
    		            :   +--------+   :
    		            :                :
    		            
每个运行在单独CPU上的程序都会执行内存操作.对一个抽象CPU,内存操作的执行次序是非常宽松的,在能保证程序上下文逻辑关系的前提下,CPU可以以任意次序执行这些操作.同样,对编译器来说,在不影响程序输出结果的前提下,编译器可以以任意次序对指令重排序.

在上面的图示中,一个CPU执行内存操作产生的影响,一直要到该操作穿越该CPU与系统中其他部分的界面(见图中的虚线)之后,才能被其他部分所察觉到.

例如考虑如下操作序列:

	CPU 1		      CPU 2
	===============	===============
	     { A == 1; B == 2 }
	A = 3;		     x = B;
	B = 4;		     y = A;       
         	

对于抽象模型中央的内存系统来说,它接收到的内存操作顺序可以被排列成如下24种不同的组合:

	STORE A=3,	STORE B=4,	y=LOAD A->3,	x=LOAD B->4
	STORE A=3,	STORE B=4,	x=LOAD B->4,	y=LOAD A->3
	STORE A=3,	y=LOAD A->3,	STORE B=4,	x=LOAD B->4
	STORE A=3,	y=LOAD A->3,	x=LOAD B->2,	STORE B=4
	STORE A=3,	x=LOAD B->2,	STORE B=4,	y=LOAD A->3
	STORE A=3,	x=LOAD B->2,	y=LOAD A->3,	STORE B=4
	STORE B=4,	STORE A=3,	y=LOAD A->3,	x=LOAD B->4
	STORE B=4, ...
	...

因此导致了4种可能的输出结果:

	x == 2, y == 1
	x == 2, y == 3
	x == 4, y == 1
	x == 4, y == 3

另外,一个CPU向内存系统提交一系列store操作产生的效果,不一定能被另一个cpu所提交的一系列load操作以store操作相同的次序察觉到.

作为这种情况的例子,让我们考虑如下操作序列:

	CPU 1		      CPU 2
	===============	===============
	{ A == 1, B == 2, C = 3, P == &A, Q == &C }
	B = 4;		      Q = P;
	P = &B		      D = *Q;
	           	           
这里存在一个明显的数据依赖,载入到D中的数据依赖于CPU2执行`Q=P`时,P所指向的内存地址.当这些操作执行完之后,下面的任何一组结果都是可能的:

	(Q == &A) and (D == 1)
	(Q == &B) and (D == 2)
	(Q == &B) and (D == 4)
	
注意,CPU2决不会将D载入到C中,因为CPU保证首先执行`Q=P`.

##操作外设

有些外设将它的控制接口以一组内存地址的方式展现(例如控制寄存器),但是访问控制寄存器的顺序却是至关紧要的.	例如,假设一个有一组内部寄存器的网卡,这些内部寄存器通过一个地址端口寄存器(A)和一个数据端口寄存器(D)来访问.可以通过执行如下代码访问内部寄存器5:

    *A = 5;
    x = *D;
	
但是上面的操作次序可以被重排成以下任一序列:

    STORE *A = 5, x = LOAD *D
    x = LOAD *D, STORE *A = 5
	
显然,只有第一个次序是正确的.第2个将会产生错误,因为它在设置就寄存器的编号之前就尝试访问寄存器了.

##担保

以下是CPU可以提供的最低担保:

* 对任意一个CPU,它所发起的有依赖关系的访存操作会被按序发送到内存系统,这意味对于:
    
        ACCESS_ONCE(Q) = P;
        smp_read_barrier_depends();
        D = ACCESS_ONCE(*Q);

    一定会以如下序列执行:
    
        Q = LOAD P, D = LOAD *Q
    
    在多数系统上,`smp_read_barrier_depends()`什么也不做,但它在DEC Alpha上是必须的.ACCESS_ONCE()则用于防止编译器乱序.注意通常你应该使用类似`rcu_dereference()`的调用来替代`smp_read_barrier_depends()`.
    
* 对给定的CPU,重叠的load和store操作保证按序执行,这意味对于:

     	a = ACCESS_ONCE(*X); ACCESS_ONCE(*X) = b;

    一定会以如下序列执行:
    
        a = LOAD *X, STORE *X = b
    
    而对于:
    
        ACCESS_ONCE(*X) = c; d = ACCESS_ONCE(*X);
	
    一定会以如下序列执行:
    
        STORE *X = c, d = LOAD *X
	
	(如果load和store的操作目标是同一个内存地址则它们被称为重叠的)
	
还有一些事情是必须被假定或者不能被假定的:

* 对于没有用`ACCESS_ONCE()`保护的访存操作,不能假定编译器编译出来的指令与代码顺序一致.

* 不能假定一系列独立的load和store操作会以代码顺序执行,这意味着对于:

        X = *A; Y = *B; *D = Z;
	
	以下任意一组操作序列都是可能:
	
        X = LOAD *A,  Y = LOAD *B,  STORE *D = Z
        X = LOAD *A,  STORE *D = Z, Y = LOAD *B
        Y = LOAD *B,  X = LOAD *A,  STORE *D = Z
        Y = LOAD *B,  STORE *D = Z, X = LOAD *A
        STORE *D = Z, X = LOAD *A,  Y = LOAD *B
        STORE *D = Z, Y = LOAD *B,  X = LOAD *A
	
* 必须假定重叠的内存访问被合并或丢弃,这意味对于:

    	X = *A; Y = *(A + 4);

    以下任意一组操作序列都是可能的: 		

        X = LOAD *A; Y = LOAD *(A + 4);
        Y = LOAD *(A + 4); X = LOAD *A;
        {X, Y} = LOAD {*A, *(A + 4) };

    而对于:
	
        *A = X; *(A + 4) = Y;
	    
    以下任意一组操作序列都是可能出现: 
    
        STORE *A = X; STORE *(A + 4) = Y;
        STORE *(A + 4) = Y; STORE *A = X;
        STORE {*A, *(A + 4) } = {X, Y};
	
	
#2 什么是内存屏障?

如上面介绍的,独立的内存操作会被cpu以随机的次序执行,在cpu之间或cpu与I/O设备交互的情况下,这种乱序执行会产生问题.我们需要一种干预手段去指示编译器和CPU,必须按我们指定的顺序执行操作.

内存屏障就是这样的干预手段.它强制屏障两端的内存操作被部分有序的执行.(译注:部分有序意味着
	
    STORE A;
    STORE B;
    write barrier;
    STORE C;
    STORE D;
	
`STORE A`和`STORE B`必定先于`STORE C`和`STORE D`执行,但A,B之间和C,D之间的执行顺序则不能保证).

这种强制的顺序非常重要,因为CPU和系统中其它设备可以使用多种手段用于提高性能,这些手段包括:指令重排,指令延迟执行,合并访存操作,内存预读,分支预测以及各种不同的缓存.内存屏障用于禁止这些手段,使得代码可以稳健的控制cpu之间以及cpu与外设之间的交互.


##内存屏障的种类:

* 1 写(store)内存屏障.

    写屏障保证,对系统中其余的部件来说,写屏障之前的写操作必定先于屏障之后的写操作发生.

    写屏障仅保证针对STORE操作的部分有序,并不要求对读操作产生任何影响.
 
    可以将CPU看成随着时间的推移向内存系统提交一系列的写操作.在这个序列中,所有写屏障之前的写操作都出现在屏障之后的写操作前面.

    [!]注意写屏障一般总是与读屏障或数据依赖屏障配对使用;请参考章节"SMP屏障配对".


* 2 数据依赖屏障.

     数据依赖屏障是一种弱化的读屏障.假如两个读操作,第二个读操作的目标依赖于第一个读操作的结果(例如:第一个读操作读取的内容是一个内存地址,第二个读操作读取这个内存地址中存放的值),这种情况下,我们需要使用一个数据依赖屏障以确保第二个读操作的目标,也就是由第一个读操作取到的地址是最新的.

     数据依赖屏障仅保证针对相互依赖的LOAD操作的部分有序,并不要求它对写操作,独立的读操作或重叠读操作产生影响.

     如在(1)中提到的,系统中的其它CPU可被视作向内存系统提交一系列写操作,当前CPU最终会察觉这些操作产生的效果.当CPU发出数据依赖屏障,那么可以保证,对屏障前任何一个读操作,如果它的目标被其它CPU操作序列中任何一个写操作改变,那么当屏障完成的时候,在这个操作序列中,那个写操作之前的所有写操作产生的效果都会被数据依赖屏障之后的读操作察觉到.
     
     请参考“内存屏障序列的示例”章节中图示的排序约束.

     [!]请注意第一个读操作和第二个操作之间确实是数据依赖而不是控制依赖.如果第二个读操作的内存地址依赖于第一个读操作,但仅用作条件判断,而不是直接访问那个地址,那么这是一种控制依赖.此时需要的是完全读屏障,甚至更严的屏障.请参考"数据依赖屏障".

     [!] 注意数据依赖屏障通常与写屏障配对使用;请参考章节"SMP屏障配对".

* 3 读(load)内存屏障.
    
    读屏障是一个数据依赖屏障,同时保证,对系统中其余的部件来说,读屏障之前的读操作必定先于屏障之后的读操作发生.
    
    读屏障仅保证针对LOAD操作的部分有序,并不要求对写操作产生任何影响.
    
    读屏障隐含了数据依赖屏障,所以用来替代数据依赖屏障.
    
    [!]注意读屏障一般总是与写屏障配对使用;请参考章节"SMP屏障配对".

* 4 通用内存屏障.

    通用内存屏障保证,对于系统中的其余部件来说,屏障之前的读和写操作必定先与屏障之后的读和写操作发生.
    
    通用内存屏障保证针读和写操作的部分有序.
    
    通用内存屏障隐含了读和写屏障,所以可以用来替代它们.
    
两个隐式类型:    
    
* 5 ACQUIRE操作.

    它的行为类似单向通过屏障.它保证对系统中其余的部件来说,ACQUIRE操作之后的内存操作必定在ACQUIRE之后发生.ACQUIRE包括LOCK和smp_load_acquire().

    而在ACQUIRE之前的内存操作有可能在ACQUIRE完成之后才发生.

    ACQUIRE操作应该与RELEASE操作配对使用.
    
* 6 RELEASE操作.

    它的行为类似单向通过屏障.它保证对系统中其余的部件来说,RELEASE操作之前的内存操作必定在RELEASE之前发生.RELEASE操作包括UNLOCK和smp_store_release().
    
    而在RELEASE之后的内存操作有可能在RELEASE完成之前就发生.
    
    通常使用了ACQUIRE和RELEASE操作就不再需要其它类型的内存屏障(有一个例外,请参考"MMIO写屏障").另外,RELEASE+ACQUIRE的配对操作不保证它的行为与一个完全内存屏障相似.但是,对一个变量执行ACQUIRE操作之后,可以保证,在对这个变量执行RELEASE之前的访存操作在执行RELEASE之后都可见.换句话说,在一个变量的临界区内,所有在临界区之前的对那个变量的访存操作保证已经完成.
    
    这意味,ACQUIRE实现了最小的获得操作语义,而RELEASE实现了最小的释放操作语义.

只有在存在CPU之间交互或CPU与外设之间交互的情况下才需要使用内存屏障.如果某段代码保证没有类似交互的情况,那么在这段代码中不需要使用任何内存屏障.

请注意,以上都是最低担保.不同的体系结构可能提供更多的保证, 但是在特定体系结构的代码之外, 不能依赖于这些额外的保证.

##对于内存屏障不能做什么假设?

以下是Linux内核的内存屏障不能提供的担保:

* 不能保证内存屏障之前的访问存操作在内存屏障完的时已经完成.内存屏障可以被想象成在CPU的访存操作队列中划了一条线,使得在这条线两侧指定类型的访存操作不能互相跨越.

* 不能保证由一个CPU发出的内存屏障会对其它CPU或硬件设备产生直接的影响.它只能间接影响第二个CPU对第一个CPU访存操作产生效果的感知顺序,但是请看下一条:

* 不能保证一个CPU对第二个CPU访存操作产生效果的感知顺序与第二个CPU发出的操作顺序一致,即使第二个CPU使用了内存屏障,除非,第一个CPU也使用了配对的内存屏障(请参考"SMP屏障配对").

* 不能保证与CPU相关的某些硬件设备不会对访存操作重排序.缓存一致性机制应该在CPU之间传递内存屏障产生的间接影响,但不保证有序的.

     [*]  要想了解总线控制DMA和一致性请阅读: 
            
     Documentation/PCI/pci.txt
     
     Documentation/DMA-API-HOWTO.txt
     
     Documentation/DMA-API.txt
         
##数据依赖屏障

对数据依赖屏障的需求有点微妙,并不总能明显的发现需要使用数据依赖屏障.作为示例,请考虑以下事件序列:

        	CPU 1		            CPU 2
        	===============	      ===============
        	{ A == 1, B == 2, C = 3, P == &A, Q == &C }
        	B = 4;
        	<write barrier>
        	ACCESS_ONCE(P) = &B
        			                 Q = ACCESS_ONCE(P);
        			                 D = *Q;

这里存在一个很明显的数据依赖关系 ,在这些事件最后,Q要么等于&A要么等于&B,也就是:

        	(Q == &A) implies (D == 1)
        	(Q == &B) implies (D == 4)
    
但是,CPU2可能先感知到了P变化之后才感知到B变化,这导致了以下情形:

        (Q == &B) and (D == 2) ????
        
这看起来似乎是一个一致性错误或逻辑关系错误, 但其实不是, 并且在一些真实的CPU中就能看到这样的行为(就比如DEC Alpha). 

为了处理这样的问题,可以将一个数据依赖或者更强的屏障插入到取地址和取数据之间:

        	CPU 1		            CPU 2
        	===============	      ===============
        	{ A == 1, B == 2, C = 3, P == &A, Q == &C }
        	B = 4;
        	<write barrier>
        	ACCESS_ONCE(P) = &B
        			                 Q = ACCESS_ONCE(P);
        			                 <data dependency barrier>
        			                 D = *Q;

这样保证Q和D只能是前面的两种情形之一,而不可能出现第三种情形.

[!]注意这种违法直觉的情况在cache分列的机器上非常容易出现.例如,一个cache处理行号为奇数cache line,另一个cache处理行号为偶数的cache line.指针P可能存放在奇数号的cache line中,而变量B则存放在偶数号的cache line中.那么如果处理偶数号cache line的cache非常繁忙,而处理奇数行号cache line的cache空闲,就会出现P==&B,而B == 2.

另一种需要数据依赖屏障的情况是,从内存中读入一个数字,然后用这个数字作为下标访问数组:

        	CPU 1		            CPU 2
        	===============	      ===============
        	{ M[0] == 1, M[1] == 2, M[3] = 3, P == 0, Q == 3 }
        	M[1] = 4;
        	<write barrier>
        	ACCESS_ONCE(P) = 1
        			                 Q = ACCESS_ONCE(P);
        			                 <data dependency barrier>
        			                 D = M[Q];

数据依赖屏障对RCU系统而言非常重要.例如,include/linux/rcupdate.h中的rcu_assign_pointer()和rcu_dereference().这两个函数可以使得将RCU指针指向新对象的时候,不会出现新指向的对象没有完全初始化的情况. 

更详尽的例子请参阅"Cache一致性"章节. 


##控制依赖

控制依赖需要的是一个完全的读屏障而不仅仅是一个数据依赖屏障.考虑以下代码片段:

        	q = ACCESS_ONCE(a);
        	if (q) {
        		<data dependency barrier>  /* BUG: No data dependency!!! */
        		p = ACCESS_ONCE(b);
        	}
        	
这无法满足需求,因为这里存在的不是数据依赖而是控制依赖,CPU可能通过预测结果将if(q)短路,从而使得在其它CPU看来,load b发生于load a的前面(译注:也就是预先判断了q为真,从而将load b放在load a之前执行).在这种情况下我们实际需要的是一个读屏障:

        	q = ACCESS_ONCE(a);
        	if (q) {
        		<read barrier>
        		p = ACCESS_ONCE(b);
        	}
        	
然而,store操作是无法预测的.所以以下例子保证了按次序执行:

        	q = ACCESS_ONCE(a);
        	if (q) {
        		ACCESS_ONCE(b) = p;
        	}
        	
请注意ACCESS_ONCE()是必要的!没有ACCESS_ONCE()多个load a可能被合并成一个,多个store b也可能被合并成一个.这可能会产生违反直觉的执行顺序.

更糟糕的是,如果编译器判断出变量a不可能为0,它可能会把if判断去掉,把以上代码优化如下:

        	q = a;
        	b = p;  /* BUG: Compiler and CPU can both reorder!!! */
        	
所以千万别漏了ACCESS_ONCE().
 
以下代码尝试强制if语句中两个分支的store操作能按序执行:

        	q = ACCESS_ONCE(a);
        	if (q) {
        		barrier();
        		ACCESS_ONCE(b) = p;
        		do_something();
        	} else {
        		barrier();
        		ACCESS_ONCE(b) = p;
        		do_something_else();
        	}        	

不幸的是,现代编译器在高优化等级下会把上面的代码转换成如下形式:

        	q = ACCESS_ONCE(a);
        	barrier();
        	ACCESS_ONCE(b) = p;  /* BUG: No ordering vs. load from a!!! */
        	if (q) {
        		/* ACCESS_ONCE(b) = p; -- moved up, BUG!!! */
        		do_something();
        	} else {
        		/* ACCESS_ONCE(b) = p; -- moved up, BUG!!! */
        		do_something_else();
        	}

这样在load a和store b之间就没有了条件语句,CPU可以将它们乱序执行:而在这里条件语句是无法被优化掉的,即使应用再高的优化等级.因此,在这样的情况下,如果要保证执行顺序,你需要显式使用内存屏障,例如,smp_store_release():

        	q = ACCESS_ONCE(a);
        	if (q) {
        		smp_store_release(&b, p);
        		do_something();
        	} else {
        		smp_store_release(&b, p);
        		do_something_else();
        	}
        	
总之,在没有显式使用内存屏障的情况下,一个if语句能保证按序执行的条件是,两个分支中的store操作是不同的,例如:

        	q = ACCESS_ONCE(a);
        	if (q) {
        		ACCESS_ONCE(b) = p;
        		do_something();
        	} else {
        		ACCESS_ONCE(b) = r;
        		do_something_else();
        	}

第一个ACCESS_ONCE()依旧是必须的,它可以防止编译器预测a的值.

另外,你必须谨慎分析局部变量q是如何被使用的,否则,编译器可能会对值做预测,然后再次把条件语句优化掉,例如:

        	q = ACCESS_ONCE(a);
        	if (q % MAX) {
        		ACCESS_ONCE(b) = p;
        		do_something();
        	} else {
        		ACCESS_ONCE(b) = r;
        		do_something_else();
        	}
        	
如果MAX == 1,那么编译器预测出(q % MAX) == 0,它就会把上面的代码转换成如下形式:

        	q = ACCESS_ONCE(a);
        	ACCESS_ONCE(b) = p;
        	do_something_else();
        	
经过这样的转换,CPU就可以将load a和store b乱序执行.尝试加入barrier()不会有任何帮助.条件语句已经被移除,barrier无法把它重新加上.因此,如果你依赖执行顺序,你需要确保MAX大于1:

        	q = ACCESS_ONCE(a);
        	BUILD_BUG_ON(MAX <= 1); /* Order load from a with store to b. */
        	if (q % MAX) {
        		ACCESS_ONCE(b) = p;
        		do_something();
        	} else {
        		ACCESS_ONCE(b) = r;
        		do_something_else();
        	}
        	
请再次注意两个分支中的store b是不一样的.如果它们一样,如先前提到过的,编译器会把它移到if语句的前面去.

你必须小心谨慎,不要过多依赖布尔短路表达式计算.考虑如下示例:

        	q = ACCESS_ONCE(a);
        	if (a || 1 > 0)
        		ACCESS_ONCE(b) = 1;
        		
因为第二个条件永远为真,编译器可以把上述代码转化如下,这样就消除了控制依赖:

        	q = ACCESS_ONCE(a);
        	ACCESS_ONCE(b) = 1;
        	
这个例子强调了,你必须确保编译器无法误解你的代码.更一般的,ACCESS_ONCE()确实能强制编译器不把load遗漏掉,但却无法强制编译器使用load的结果.

最后,控制依赖是无法传递的.这通过以下两个相关的示例来展示,在这两个示例中,x和y的初始值都是0:

        	CPU 0                     CPU 1
        	=====================     =====================
        	r1 = ACCESS_ONCE(x);      r2 = ACCESS_ONCE(y);
        	if (r1 > 0)               if (r2 > 0)
        	  ACCESS_ONCE(y) = 1;       ACCESS_ONCE(x) = 1;
        
        	assert(!(r1 == 1 && r2 == 1));        	

上面的示例永远不会触发assert().但是,假如依赖控制是可传递的, 那么增加一个CPU并执行如下代码,那么断言也保证不会触发:       		        	        	        	        	              

            CPU 2
        	=====================
        	ACCESS_ONCE(x) = 2;
        
        	assert(!(r1 == 2 && r2 == 1 && x == 2)); /* FAILS!!! */
        	
但是因为控制依赖不具有传递性,所以上面示例中,第3个CPU中的断言可能会触发.如果你需要以上3CPU的例子能按序执行,你需要在CPU 0和CPU 1的load和store操作之间添加smp_mb(),也就是在if语句的前面或后面.

以上两个示例是下面这篇论文中的LB和WWC石蕊测试(立马知道结果的测试):

http://www.cl.cam.ac.uk/users/pes20/ppc-supplemental/test6.pdf and this
site: https://www.cl.cam.ac.uk/~pes20/ppcmem/index.html.

总结:

* 控制依赖可以使得前面的load和随后与这个load相关的store按序执行.但是,它不保证其它的操作也能按序执行:前面load和其后的load,以及前面store和其后的其它操作.如果你要保证其它操作的有序性请使用smb_rmb(),smp_wmb().对于前面store后跟load的情况还可以使用smp_mb().

* 如果if语句的两个分支都以对同一个变量执行store起始,那么应该在分支语句的起始处添加barrier().

* 达成控制依赖的条件是,在load和随后的store之间至少要有一个运行时条件判断语句,且那个条件判断语句使用了前面load操作的值.如果编译器把条件判断语句优化掉了,那么它也可能打乱load和store的顺序.正确使用ACCESS_ONCE()可以帮助防止必要的条件判断语句被编译器优化.

* 需要小心防止编译器优化把控制依赖消除.正确的使用ACCESS_ONCE()或barrier()可以帮助防止这种情况的发生.更多的信息请参考编译器屏障.

* 控制依赖不具备传递性.如果需要传递性请使用smp_mb().

##SMP屏障配对

对于CPU之间交互的情况,特定类型的内存屏障必须配对使用.缺乏适当配对的情况几乎总是错误的.

通用屏障之间互相配对,虽然它可以与几乎所有其它类型的屏障配对,但这会失去可传递性.获取屏障与释放屏障配对,同时它们也可以与其它屏障配对,包括通用屏障.一个写屏障与数据依赖屏障,获取屏障,释放屏障,读屏障,或通用屏障配对.类似的,一个读屏障或数据依赖屏障与写屏障,获取屏障,释放屏障或通用屏障配对:

        	CPU 1		            CPU 2
        	===============	      ===============
        	ACCESS_ONCE(a) = 1;
        	<write barrier>
        	ACCESS_ONCE(b) = 2;      x = ACCESS_ONCE(b);
        			                 <read barrier>
        			                 y = ACCESS_ONCE(a);

或:

        	CPU 1		            CPU 2
        	===============	      ===============================
        	a = 1;
        	<write barrier>
        	ACCESS_ONCE(b) = &a;     x = ACCESS_ONCE(b);
        			                 <data dependency barrier>
        			                 y = *x;

基本上, 读屏障总是需要用在这些地方的, 尽管可以使用更"弱"的类型.     	

[!] 注意,写屏障之前的store通常总是与写屏障或数据依赖屏障之后的load匹配,反之亦然:

        	CPU 1                               CPU 2
        	===================                 ===================
        	ACCESS_ONCE(a) = 1;  }----   --->{  v = ACCESS_ONCE(c);
        	ACCESS_ONCE(b) = 2;  }    \ /    {  w = ACCESS_ONCE(d);
        	<write barrier>            \        <read barrier>
        	ACCESS_ONCE(c) = 3;  }    / \    {  x = ACCESS_ONCE(a);
        	ACCESS_ONCE(d) = 4;  }----   --->{  y = ACCESS_ONCE(b);
        	
        	
##内存屏障举例

第1,写屏障可以使得store操作部分有序,考虑如下事件序列:

        	CPU 1
        	=======================
        	STORE A = 1
        	STORE B = 2
        	STORE C = 3
        	<write barrier>
        	STORE D = 4
        	STORE E = 5

这个操作序列会被有序的提交到内存一致性系统,但是系统中其它部分所感知到的顺序是无序集合{STORE A,STORE B,STORE C}中的事件先于无序集合{STORE D,STORE E}中的事件,但是各集合中事件的顺序可以是任意组合:

        	+-------+       :      :
        	|       |       +------+
        	|       |------>| C=3  |     }     /\
        	|       |  :    +------+     }-----  \  -----> Events perceptible to
        	|       |  :    | A=1  |     }        \/       the rest of the system
        	|       |  :    +------+     }
        	| CPU 1 |  :    | B=2  |     }
        	|       |       +------+     }
        	|       |   wwwwwwwwwwwwwwww }   <--- At this point the write barrier
        	|       |       +------+     }        requires all stores prior to the
        	|       |  :    | E=5  |     }        barrier to be committed before
        	|       |  :    +------+     }        further stores may take place
        	|       |------>| D=4  |     }
        	|       |       +------+
        	+-------+       :      :
        	                   |
        	                   | Sequence in which stores are committed to the
        	                   | memory system by CPU 1
        	                   V
        
 第2,数据依赖屏障可以使得有数据依赖的load操作部分有序,考虑如下事件序列:         
 
 
         	CPU 1			         CPU 2
        	=======================	=======================
        		{ B = 7; X = 9; Y = 8; C = &Y }
        	STORE A = 1
        	STORE B = 2
        	<write barrier>
        	STORE C = &B		       LOAD X
        	STORE D = 4		        LOAD C (gets &B)
        				               LOAD *C (reads B)      	
        				
在CPU2没有干预的情况下, 感知到CPU1的操作产生效果的顺序可能是随机的,尽管CPU1使用了写屏障:

        	+-------+       :      :                :       :
        	|       |       +------+                +-------+  | Sequence of update
        	|       |------>| B=2  |-----       --->| Y->8  |  | of perception on
        	|       |  :    +------+     \          +-------+  | CPU 2
        	| CPU 1 |  :    | A=1  |      \     --->| C->&Y |  V
        	|       |       +------+       |        +-------+
        	|       |   wwwwwwwwwwwwwwww   |        :       :
        	|       |       +------+       |        :       :
        	|       |  :    | C=&B |---    |        :       :       +-------+
        	|       |  :    +------+   \   |        +-------+       |       |
        	|       |------>| D=4  |    ----------->| C->&B |------>|       |
        	|       |       +------+       |        +-------+       |       |
        	+-------+       :      :       |        :       :       |       |
        	                               |        :       :       |       |
        	                               |        :       :       | CPU 2 |
        	                               |        +-------+       |       |
        	    Apparently incorrect --->  |        | B->7  |------>|       |
        	    perception of B (!)        |        +-------+       |       |
        	                               |        :       :       |       |
        	                               |        +-------+       |       |
        	    The load of X holds --->    \       | X->9  |------>|       |
        	    up the maintenance           \      +-------+       |       |
        	    of coherence of B             ----->| B->2  |       +-------+
        	                                        +-------+
        	                                        :       :
        
在上面的示例中,CPU2得到B=7(LOAD C),经管在代码顺序上LOAD *C在LOAD C的后面.

如果CPU2在LOAD C和LOAD *C之间插入一个数据依赖屏障:

        	CPU 1			          CPU 2
        	=======================	=======================
        		{ B = 7; X = 9; Y = 8; C = &Y }
        	STORE A = 1
        	STORE B = 2
        	<write barrier>
        	STORE C = &B		       LOAD X
        	STORE D = 4		        LOAD C (gets &B)
        				               <data dependency barrier>
        				               LOAD *C (reads B)       				
 
 就会发生下面的事情:
 
         +-------+       :      :                :       :
        	|       |       +------+                +-------+
        	|       |------>| B=2  |-----       --->| Y->8  |
        	|       |  :    +------+     \          +-------+
        	| CPU 1 |  :    | A=1  |      \     --->| C->&Y |
        	|       |       +------+       |        +-------+
        	|       |   wwwwwwwwwwwwwwww   |        :       :
        	|       |       +------+       |        :       :
        	|       |  :    | C=&B |---    |        :       :       +-------+
        	|       |  :    +------+   \   |        +-------+       |       |
        	|       |------>| D=4  |    ----------->| C->&B |------>|       |
        	|       |       +------+       |        +-------+       |       |
        	+-------+       :      :       |        :       :       |       |
        	                               |        :       :       |       |
        	                               |        :       :       | CPU 2 |
        	                               |        +-------+       |       |
        	                               |        | X->9  |------>|       |
        	                               |        +-------+       |       |
        	  Makes sure all effects --->   \   ddddddddddddddddd   |       |
        	  prior to the store of C        \      +-------+       |       |
        	  are perceptible to              ----->| B->2  |------>|       |
        	  subsequent loads                      +-------+       |       |
        	                                        :       :       +-------+
        
第3,读屏障可以使得load操作部分有序,考虑如下事件序列:

        	CPU 1			          CPU 2
        	=======================	=======================
        		{ A = 0, B = 9 }
        	STORE A=1
        	<write barrier>
        	STORE B=2
        				               LOAD B
        				               LOAD A

在CPU2没有干预的情况下, 感知到CPU1的操作产生效果的顺序可能是随机的,尽管CPU1使用了写屏障:

        	+-------+       :      :                :       :
        	|       |       +------+                +-------+
        	|       |------>| A=1  |------      --->| A->0  |
        	|       |       +------+      \         +-------+
        	| CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
        	|       |       +------+        |       +-------+
        	|       |------>| B=2  |---     |       :       :
        	|       |       +------+   \    |       :       :       +-------+
        	+-------+       :      :    \   |       +-------+       |       |
        	                             ---------->| B->2  |------>|       |
        	                                |       +-------+       | CPU 2 |
        	                                |       | A->0  |------>|       |
        	                                |       +-------+       |       |
        	                                |       :       :       +-------+
        	                                 \      :       :
        	                                  \     +-------+
        	                                   ---->| A->1  |
        	                                        +-------+
        	                                        :       :


如果CPU2在LOAD B和LOAD A之间插入一个读屏障:

        	CPU 1			          CPU 2
        	=======================	=======================
        		{ A = 0, B = 9 }
        	STORE A=1
        	<write barrier>
        	STORE B=2
        				               LOAD B
        				               <read barrier>
        				               LOAD A               	
        				
CPU2就可以以正确的顺序感知到CPU1的操作所产生的效果:

        	+-------+       :      :                :       :
        	|       |       +------+                +-------+
        	|       |------>| A=1  |------      --->| A->0  |
        	|       |       +------+      \         +-------+
        	| CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
        	|       |       +------+        |       +-------+
        	|       |------>| B=2  |---     |       :       :
        	|       |       +------+   \    |       :       :       +-------+
        	+-------+       :      :    \   |       +-------+       |       |
        	                             ---------->| B->2  |------>|       |
        	                                |       +-------+       | CPU 2 |
        	                                |       :       :       |       |
        	                                |       :       :       |       |
        	  At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
        	  barrier causes all effects      \     +-------+       |       |
        	  prior to the storage of B        ---->| A->1  |------>|       |
        	  to be perceptible to CPU 2            +-------+       |       |
        	                                        :       :       +-------+        				

为了更完全的说明,考虑如下在两个LOAD A之间插入一个读屏障的示例:

        	CPU 1			          CPU 2
        	=======================	=======================
        		{ A = 0, B = 9 }
        	STORE A=1
        	<write barrier>
        	STORE B=2
        				              LOAD B
        				              LOAD A [first load of A]
        				              <read barrier>
        				              LOAD A [second load of A]        							
        				
虽然两个LOAD A都发生在LOAD B的后面,他们还是有可能得到不一样的值:

        	+-------+       :      :                :       :
        	|       |       +------+                +-------+
        	|       |------>| A=1  |------      --->| A->0  |
        	|       |       +------+      \         +-------+
        	| CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
        	|       |       +------+        |       +-------+
        	|       |------>| B=2  |---     |       :       :
        	|       |       +------+   \    |       :       :       +-------+
        	+-------+       :      :    \   |       +-------+       |       |
        	                             ---------->| B->2  |------>|       |
        	                                |       +-------+       | CPU 2 |
        	                                |       :       :       |       |
        	                                |       :       :       |       |
        	                                |       +-------+       |       |
        	                                |       | A->0  |------>| 1st   |
        	                                |       +-------+       |       |
        	  At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
        	  barrier causes all effects      \     +-------+       |       |
        	  prior to the storage of B        ---->| A->1  |------>| 2nd   |
        	  to be perceptible to CPU 2            +-------+       |       |
        	                                        :       :       +-------+

但是CPU 2也可能在读屏障结束之前就感知到CPU 1对A的更新:

        	+-------+       :      :                :       :
        	|       |       +------+                +-------+
        	|       |------>| A=1  |------      --->| A->0  |
        	|       |       +------+      \         +-------+
        	| CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
        	|       |       +------+        |       +-------+
        	|       |------>| B=2  |---     |       :       :
        	|       |       +------+   \    |       :       :       +-------+
        	+-------+       :      :    \   |       +-------+       |       |
        	                             ---------->| B->2  |------>|       |
        	                                |       +-------+       | CPU 2 |
        	                                |       :       :       |       |
        	                                 \      :       :       |       |
        	                                  \     +-------+       |       |
        	                                   ---->| A->1  |------>| 1st   |
        	                                        +-------+       |       |
        	                                    rrrrrrrrrrrrrrrrr   |       |
        	                                        +-------+       |       |
        	                                        | A->1  |------>| 2nd   |
        	                                        +-------+       |       |
        	                                        :       :       +-------+


这保证了,在CPU2上,如果LOAD B得到B==2那么对第二个LOAD A一定会得到A==1.但对第一个LOAD A则没有这种保证;所以第一个LOAD可能得到A==0或A==1.

##读内存屏障VS内存预取

很多CPU会对LOAD操作进行内存预取:CPU提前发现需要从内存中读入数据,并且发现总线空闲(没有其他LOAD操作),于是CPU提前将数据载入,尽管指令的执行流还没到达LOAD的点上.这使得LOAD操作可能立即完成,因为CPU已经提前获得了要处理的数据.    

最终被预取的值可能没机会使用,例如load所在的分支没被执行,这种情况下预取的值可能会直接丢弃,也可能被缓存供以后使用.

考虑:

        	CPU 1			CPU 2
        	=======================	=======================
        				LOAD B
        				DIVIDE		} Divide instructions generally
        				DIVIDE		} take a long time to perform
        				LOAD A
        
看上去可能是这样:

        	                                        :       :       +-------+
        	                                        +-------+       |       |
        	                                    --->| B->2  |------>|       |
        	                                        +-------+       | CPU 2 |
        	                                        :       :DIVIDE |       |
        	                                        +-------+       |       |
        	The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
        	division speculates on the              +-------+   ~   |       |
        	LOAD of A                               :       :   ~   |       |
        	                                        :       :DIVIDE |       |
        	                                        :       :   ~   |       |
        	Once the divisions are complete -->     :       :   ~-->|       |
        	the CPU can then perform the            :       :       |       |
        	LOAD with immediate effect              :       :       +-------+


在第2个load之前放置一个读屏障或数据依赖屏障:

        	CPU 1			CPU 2
        	=======================	=======================
        				LOAD B
        				DIVIDE
        				DIVIDE
        				<read barrier>
        				LOAD A            
        				
会强制依据使用的屏障类型来重新考虑如何使用预取的值.如果被预取的内存没有发生变化,那么预取的值就会被采用:

        	                                        :       :       +-------+
        	                                        +-------+       |       |
        	                                    --->| B->2  |------>|       |
        	                                        +-------+       | CPU 2 |
        	                                        :       :DIVIDE |       |
        	                                        +-------+       |       |
        	The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
        	division speculates on the              +-------+   ~   |       |
        	LOAD of A                               :       :   ~   |       |
        	                                        :       :DIVIDE |       |
        	                                        :       :   ~   |       |
        	                                        :       :   ~   |       |
        	                                    rrrrrrrrrrrrrrrr~   |       |
        	                                        :       :   ~   |       |
        	                                        :       :   ~-->|       |
        	                                        :       :       |       |
        	                                        :       :       +-------+
        				
  
  但是如果有来自其他CPU的更新或失效,那么预取的值将会被丢弃,并从内存中重新读取:

		                                        :       :       +-------+
	                                        +-------+       |       |
	                                    --->| B->2  |------>|       |
	                                        +-------+       | CPU 2 |
	                                        :       :DIVIDE |       |
	                                        +-------+       |       |
	The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
	division speculates on the              +-------+   ~   |       |
	LOAD of A                               :       :   ~   |       |
	                                        :       :DIVIDE |       |
	                                        :       :   ~   |       |
	                                        :       :   ~   |       |
	                                    rrrrrrrrrrrrrrrrr   |       |
	                                        +-------+       |       |
	The speculation is discarded --->   --->| A->1  |------>|       |
	and an updated value is                 +-------+       |       |
	retrieved                               :       :       +-------+

##传递性

传递性是对有序性非常直观的概念,真实的计算机系统通常不支持传递性.下面的例子展示了什么是传递性(或称为累积性):

        	CPU 1			CPU 2			     CPU 3
        	==============	=================	================
        		{ X = 0, Y = 0 }
        	STORE X=1		LOAD X			    STORE Y=1
        				     <general barrier>	 <general barrier>
        				     LOAD Y			     LOAD X

假如在CPU 2上LOAD X返回1且LOAD Y返回0.这意味在某种意义上CPU 2的LOAD X跟在CPU 1的STORE X之后,且CPU 2的LOAD Y先于CPU 3的STORE Y.现在的问题是,CPU 3的LOAD X会返回0吗?

因为CPU 2的LOAD X在某种意义上跟在CPU 1的STORE之后,很自然的会认为CPU 3的LOAD X也会返回1.这种期望是传递性的一个例子:如果CPU A上执行的LOAD跟在对相同变量执行LOAD的CPU B之后,那么CPU A的LOAD必须要么返回与CPU B的LOAD一样的值,要么返回比他更新的值.

在Linux内核中,使用通用屏障可以保证传递性.因此在上面的示例中,如果在CPU 2上,LOAD X返回1且LOAD Y返回0,那么在CPU 3上LOAD X也1.

但是,读/写屏障不保证传递性.假如我们将上例中CPU 2上的通用屏障替换或一个读屏障:

	CPU 1			 CPU 2			     CPU 3
	==============	==================	=================
		      { X = 0, Y = 0 }
	STORE X=1		 LOAD X			    STORE Y=1
				     <read barrier>		 <general barrier>
				     LOAD Y			     LOAD X

这样的替换会破坏传递性:在CPU 2上,LOAD X返回1且LOAD Y返回0,在CPU 3上LOAD X返回0也是完全合法的.

关键在于,CPU 2上的读屏障使得它的两个LOAD按序执行,但不保证对CPU 1的STORE操作有任何影响.因此如果本例运行在一个CPU 1和CPU 2共享写缓冲或cache的系统上,CPU 2可能会提前访问到CPU 1写值.因此需要通用屏障确保所有CPU对CPU 1和CPU 2的访存顺序达成一致.

再次重申,如果你的代码依赖传递性,请自始至终使用通用屏障.

#内核中的显式屏障

Linux内核中存在3种不同的屏障,分别作用在不同的层次上:

* 编译器屏障

* CPU内存屏障

* MMIO写屏障

##编译器屏障

Linux内核中存在一个显式的编译器屏障,用于防止编译器将屏障一側的访存操作移到另一侧:

    barrier();
    
这是一种通用屏障,不分读写类型.但是,ACCESS_ONCE()可被视为一种更弱版本的barrier(),它仅仅作用在ACCESS_ONCE()指定的变更量上.

barrier()函数回产生如下影响:

* 防止编译器将barrier()一側的访存操作移到另一侧.一个应用例子在可以使被中断的代码和中断处理器中的代码的沟通更方便.

* 在一个循环语句中,强制编译器每次都从内存读入用作循环条件判断的变量.

ACCESS_ONCE()用于防止一些优化手段,这样的优化手段在一个单线程的运行环境下是完全安全的,但是如果把代码放在多线程环境下运行就会产生严重的错误.下面列举会产生这种问题的优化:

* 编译器有权对在同一变量上的load和store操作重排序,在某些情况下,CPU也会这么做.这意味:

        	a[0] = x;
        	a[1] = x;

   的实际执行顺序可能是`a[1] = x;a[0] = x;`.要想防止编译器或CPU执行这种重排序可以象下面这样:

        	a[0] = ACCESS_ONCE(x);
        	a[1] = ACCESS_ONCE(x);

   简单来说,ACCESS_ONCE()为多个CPU访问同一变量提供了cache一致性.

* 编译器有权将两个对同一变量的load操作合并.编译器可以如下代码:

        	while (tmp = a)
        		do_something_with(tmp);
        		
   优化成,虽然在某种意义上是单线程安全的,但显然违背开发者意图的代码:

        	if (tmp = a)
        		for (;;)
        			do_something_with(tmp);

   使用ACCESS_ONCE()可以防止编译器做这样的优化:

        	while (tmp = ACCESS_ONCE(a))
        		do_something_with(tmp);

* 编译器有权重载一个变量,例如,寄存器紧缺会阻碍编译器将它感兴趣的数据都保存在寄存器中.所以编译器可能因此将tmp变量给优化掉:

        	while (tmp = a)
        		do_something_with(tmp);

   这会产生如下代码,在单线程执行的时候是完全安全的,但在多线程并发执行的情况下会导致严重错误:

        	while (a)
        		do_something_with(a);
        		
   例如,假如a是一个在多线程之间共享的变量,如果在while和调用do_something_with()之间,a被其它线程设置为0,以上的优化代码就会出现将0传给do_something_with()的情况.

   再一次,我们使用ACCESS_ONCE()放置编译器做这样的优化:

        	while (tmp = ACCESS_ONCE(a))
        		do_something_with(tmp);        		

   注意,如果编译器发现寄存器不足,它会把tmp保存在栈上.保存和之后的载入产生会开销是编译器选择重载变量的原因.这对于单线程代码是安全的,所以你  需要告诉编译器在什么情况下这样是不安全的,阻止它做类似的优化.

* 如果编译器可以提前预知一个值,那么它有权将相关的load操作删除.例如,如果编译器可以预测a永远是0,那么它就会将如下代码优化:

        	while (tmp = a)
        		do_something_with(tmp);  				
				
    成:
    
        do { } while (0);
        
    这个优化对单线程代码来说非常有效,它减少了一个load和一个分支判断.但如果a是一个在多线程间共享的变量,就会出现严重的问题.使用ACCESS_ONCE()可以防止编译器做类似优化:
    
            	while (tmp = ACCESS_ONCE(a))
        		  do_something_with(tmp);          								

    但请注意,编译器会密切关注你是如何使用用ACCESS_ONCE()载入的值.例如,假设你写下如下代码,且MAX是被定义为1的宏变量:
    
            	while ((tmp = ACCESS_ONCE(a)) % MAX)
        		  do_something_with(tmp);
        		  
    那么,编译器知道对任何数应用%都会导致结果为0,那边编译器又可以把这段代码优化掉了.(与上一段被优化的代码的区别在,load a还是执行的)
    
* 类似的,如果编译器发现对一个变量执行store时,要写进去的值是那个变量已经持有的,那么它有权将这个store操作删除.再一次,编译器假设当前CPU是唯一一个对那个变量执行store的CPU,这样的假设会使得编译器对一个共享变量做出错误的优化决策.例如,假设你写了如下代码:

        	a = 0;
        	/* Code that does not store to variable a. */
        	a = 0;
        	
    编译器发现a的值已经是0了,所以他把第二个store给删除了.            	
                 		  