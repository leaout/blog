---
title: pg数据库死锁检测
date: 2024-12-17
tags: 数据库
---
## 概念

每个事务都在等待集合中的另一事务，由于这个集合是一个有限集合，因此一旦在这个等待的链条上产生了环，就会产生死锁。自旋锁和轻量锁属于系统锁，他们目前没有死锁检测机制，只能靠内核开发人员在开发过程中谨慎的使用锁，避免死锁发生。  
数据库中的常规锁，对申请锁的顺序没有严格的限制，在申请锁时也没有严格的查验，因此不可避免的就会产生死锁。pg数据库采用一种全局死锁检测的方法就是，每个进程启动后会启动一个定时器，定时调用死锁检测函数，检测是否有死锁产生，如果有死锁则进行相应的处理来解除死锁。

### 实边

在常规锁的申请过程中，假设A事务持有表的共享锁或排它锁，当时事务B申请表的排他锁时，就需要进入等待状态，即有等待状态B->A,我们称这种等待边为实边。它主要出现在等待者和持锁者之间。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/73cd1dcb30c84198adb1055d2eee7588.png#pic_center)

### 虚边

假设事务A持有共享锁，事务B要申请排他锁，因为与事务A冲突，那么事务B就需要进入等待队列，如果此时又有事务C要申请共享锁，虽然事务C与事务A并不冲突，但是事务C与等待队列中的事务B要持有的排他锁冲突，所以事务C也要进入等待队列中，此时的冲突关系C->B ,我们称他们之间等待的边为虚边。它主要出现在等待队列的等待者之间的关系。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0e90a10038d94393b5b94a7b1bf460cb.png#pic_center)

### 环

在数据库中，假设事务A需要等待事务B释放锁后才能执行，这样的等待关系我们称之为边，边是有向的。实边和虚边是边的两种特殊情况。假设一个锁的等待队列中的所有的事务，他们之间的等待关系都用边来表示，这样构成关系图实际上就是一种有向图。如果有向图中出现了环，我们就可以判断出现了死锁。如果环的每个边都是实边，那么就是出现了实边死锁，实边死锁只能通过杀掉其中一个进程来断开环从而解锁。 如果环中存在虚边，那么出现的就是虚边死锁，就可以通过调整等待队列的顺序来尝试断开环，从而解锁。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1ca408dca106471fb6968623358dd2a8.png#pic_center)

+   假设有A、B、C三个事务，有Lock1、Lock2两把锁，其中事务A等待Lock1的排它锁；事务B持有Lock1的共享锁，等待Lock2的共享锁；事务C持有Lock 2的排它锁，等待Lock1的共享锁。
+   对于Lock1来说，事务B持有其共享锁，事务A等待Lock1的排它锁，与事务B形成实边，事务C等待Lock1的共享锁，与事务B并不冲突，但是与事务A冲突，所以与事务A形成虚边。
+   对于Lock2来说，事务C持有Lock2的排它锁，事务B在等待Lock2的共享锁，与事务C冲突，形成实边。
+   最终得到的等待关系图，如上图，形成了一个包含虚边的环，即虚边死锁；即事务A在等待事务B释放Lock1，事务B在等待事务C释放Lock2，而事务C在等待事务A释放Lock1，从而形成了死锁
+   要解除上面的虚边死锁，可以对虚边进行拓扑排序，调整虚边的等待关系，即将Lock1等待队列上的C–>A,改成A–>C,这样得到的等待关系图，就不存在环了，死锁就不存在了。事务A在等待事务C释放Lock1，事务B在等待事务C释放Lock2，事务C可以直接获取Lock1的共享锁，执行完之后就可以释放Lock2的排他锁，释放后，事务A获取Lock1锁，事务B获取Lock2锁，都不再阻塞。  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b9c5333176aa41228754a473e8a1f086.png#pic_center)

### 拓扑排序

当存在虚边构成的环时，会通过重排等待队列的方式尝试断开环从而解锁。重排的方式就是通过拓扑排序实现。  
拓扑排序就是在一个有向无环图（DAG），将所有的点 排成一个线性的序列，使得每条有向边的起点都排在终点的前面。  
拓扑排序遵循的原理：  
1） 在图中选择一个没有前驱的定点V  
2） 从图中删除顶点V和所有以该顶点为尾的弧。  
如下图：  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c594a1f5ee2647e88bd33e6866c68d35.png#pic_center)

1.  找到没有前驱的定点V1
2.  删除V1及以V1作为起点的边
3.  继续查找没有前驱的顶点，此时V2和V3都符合要求，随机选择一个，这里选择V2
4.  删除V2和以V2作为起点的边
5.  继续查找没有前驱的顶点，V3符合，选择V3
6.  删除V3以及以V3作为起点的边
7.  剩余V4，排序结束。  
    最终得到的拓扑排序结果就有两种：  
    V1->V2->V3->V4  
    V1->V3->V2->V4  
    死锁检测函数中调用TopoSort函数实现对包含虚边的环的拓扑排序。

## 死锁检测相关的结构体和全局变量

### 结构体

#### EDGE

等待关系图中的一条边。  
等待者（waiter）和阻塞者（blocker）可能是锁组的成员，也可能不是，但如果它们中任何一个属于锁组，那它将是锁组的领导者而非锁组中的其他成员。即便这些特定进程根本无需等待，锁组的领导者也充当整个组的代表。等待者的锁组中至少有一个成员在给定锁的等待队列上，甚至可能更多。

```c
typedef struct
{
	PGPROC	   *waiter;			/* 等待者*/
	PGPROC	   *blocker;		/* 被等待者or */
	LOCK	   *lock;			/* 等待的锁*/
	int			pred;			/* 拓扑排序使用的额外变量 */
	int			link;			/* 拓扑排序使用的额外变量*/
} EDGE;
```

#### WAIT\_ORDER

等待队列，如果死锁检测处有虚边死锁，则会尝试通过调整等待队列来尝试消除死锁，调整时新的等待队列就保存到waitOrders数组中，数组中每个元素就是一个等待队列，由WAIT\_ORDER结构体保存其相关信息。

```c
typedef struct
{
	LOCK	   *lock;			/* 等待的锁 */
	PGPROC	  **procs;			/* 在lock上的新的等待队列 */
	int			nProcs;         /* 等待队列的长度 */
} WAIT_ORDER;
```

#### DEADLOCK\_INFO

死锁相关的信息

```c
typedef struct
{
	LOCKTAG		locktag;		/* 死锁的锁tag信息*/
	LOCKMODE	lockmode;		/* 等待的锁的模式*/
	int			pid;			/* 阻塞的进程号*/
} DEADLOCK_INFO;
```

### 全局变量

+   got\_deadlock\_timeout： 全局死锁检测标志位，为true时触发死锁死锁检测。
+   nCurConstraints: 当前检测到的边的数量
+   curConstraints： 当前已被检测的虚边的信息，数组中保存，拓扑排序时就对这个数组进行排序。
+   maxCurConstraints： 允许的最大的边数量
+   nPossibleConstraints： 可能的边的数量
+   possibleConstraints： 可以被调整的虚边的信息，数组中保存
+   nWaitOrders： 新的等待队列的数量
+   waitOrders： 新的等待队列的信息，在查找环的过程中，会将对应的等待边放到该队列中，方便进行重新排列。大小是进程数的1/2，因为一个边就有2个等待进程。
+   blocking\_autovacuum\_proc： 被vacuum阻塞的进程
+   nVisitedProcs： 访问到的进程的数量
+   visitedProcs： 访问到的进程信息，数组中保存，通过该数组判断是否存在环，比如如果一个进程在数组中重复出现则就构成了环。
+   nDeadlockDetails： 死锁详细信息的数量
+   deadlockDetails： 死锁详细信息，数组中保存
+   beforeConstraints：记录每个进程需要在多少其他进程之前
+   afterConstraints：则间接通过链表头记录每个进程需要在哪些进程之后。

## 死锁检查流程及相关函数

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/873e511094ba46e2bcbc981c4cfa4527.png#pic_center)

### 注册死锁检测定时器

在每个进程启动时，会注册一个死锁检测定时器,回调函数为CheckDeadLockAlert，当DEADLOCK\_TIMEOUT时间超时（默认是1秒）时，就会调用CheckDeadLockAlert函数，该函数内会将全局变量got\_deadlock\_timeout设为true.

```c
	if (!bootstrap)
	{
		RegisterTimeout(DEADLOCK_TIMEOUT, CheckDeadLockAlert);
		RegisterTimeout(STATEMENT_TIMEOUT, StatementTimeoutHandler);
		RegisterTimeout(LOCK_TIMEOUT, LockTimeoutHandler);
		RegisterTimeout(IDLE_IN_TRANSACTION_SESSION_TIMEOUT,
						IdleInTransactionSessionTimeoutHandler);
		RegisterTimeout(IDLE_SESSION_TIMEOUT, IdleSessionTimeoutHandler);
		RegisterTimeout(CLIENT_CONNECTION_CHECK_TIMEOUT, ClientCheckTimeoutHandler);
	}
```

进程在等待锁时会调用WaitOnLock函数等待，该函数又回调用procsleep函数，它里面就会根据该变量判断是否需要进行死锁检测。

```c
			if (got_deadlock_timeout)
			{
				CheckDeadLock();
				got_deadlock_timeout = false;
			}
```

### InitDeadLockChecking

每个backend进程启动后调用，初始化死锁检测相关的全局变量, maxBackend为进程的最大数量（max\_connections),其中死锁检测相关的全局变量初始化的大小为：  
MaxBackends = MaxConnections + autovacuum\_max\_workers + 1 +  
max\_worker\_processes + max\_wal\_senders = 100 + 3 + 1 + 8 + 10 = 122（默认值）

| 全部变量名 | 长度 |
| --- | --- |
| visitedProcs | MaxBackends |
| deadlockDetails | MaxBackends |
| topoProcs | MaxBackends |
| beforeConstraints | MaxBackends |
| afterConstraints | MaxBackends |
| waitOrders | MaxBackends / 2 |
| waitOrderProcs | MaxBackends |
| maxCurConstraints | MaxBackends |
| curConstraints | MaxBackends |
| maxPossibleConstraints | MaxBackends \* 4 |
| possibleConstraints | MaxBackends \* 4 |

```c
	visitedProcs = (PGPROC **) palloc(MaxBackends * sizeof(PGPROC *));//访问过的进程数组，最大为maxbankend
	deadlockDetails = (DEADLOCK_INFO *) palloc(MaxBackends * sizeof(DEADLOCK_INFO));//死锁详细信息，最大为maxBackends

	/*
	 拓扑排序用
	 */
	topoProcs = visitedProcs;	/* re-use this space */
	beforeConstraints = (int *) palloc(MaxBackends * sizeof(int));
	afterConstraints = (int *) palloc(MaxBackends * sizeof(int));
	waitOrders = (WAIT_ORDER *)
		palloc((MaxBackends / 2) * sizeof(WAIT_ORDER));//等待队列数组，长度为MaxBackends/2
	waitOrderProcs = (PGPROC **) palloc(MaxBackends * sizeof(PGPROC *));//等待队列进程，最大为Maxbackends
	maxCurConstraints = MaxBackends;
	curConstraints = (EDGE *) palloc(maxCurConstraints * sizeof(EDGE));//探索的边的信息，最大为MaxBackends，设置过大会导致嵌套深度过大导致堆栈溢出

	/*
	 * 允许最多保存3*MaxBackends个约束而无需重新运行TestConfiguration。
	 （这可能已经绰绰有余，但即使空间不足，我们也可以通过每次需要时重新运行TestConfiguration来重新计算约束列表以应对。）
	 possibleConstraints[]中的最后MaxBackends个条目被预留作为FindLockCycle的输出工作区。
	 */
	maxPossibleConstraints = MaxBackends * 4;
	possibleConstraints =
		(EDGE *) palloc(maxPossibleConstraints * sizeof(EDGE));
```

### CheckDeadLock

这是死锁检测的入口函数，死锁检测操作就是由该函数实现。由于死锁检测是互斥的，所以死锁检测期间锁表不允许被修改。但是如果一个事务只是通过本地锁表或通过FastPath就能获得锁，则它不受死锁检测的影响。

+   以排他模式锁住主锁表

```c
	for (i = 0; i < NUM_LOCK_PARTITIONS; i++)
		LWLockAcquire(LockHashPartitionLockByIndex(i), LW_EXCLUSIVE);//以排他模式锁住主锁表
```

+   调用DeadLockCheck函数检测死锁

```c
deadlock_state = DeadLockCheck(MyProc);//检测死锁
```

+   如果产生了实边死锁，将当前进程从等待队列中删除

```c
if (deadlock_state == DS_HARD_DEADLOCK)//产生实边死锁
	{
			RemoveFromWaitQueue(MyProc, LockTagHashCode(&(MyProc->waitLock->tag)));//将当前进程从等待队列中删除
	}
```

+   释放主锁表

```c
	for (i = NUM_LOCK_PARTITIONS; --i >= 0;)
		LWLockRelease(LockHashPartitionLockByIndex(i));//释放所有的锁
```

### DeadLockCheck

检查给定的进程上是否产生了死锁，如果是虚边死锁，会通过调整等待队列顺序来尝试解决死锁，如果无法解决，就会返回DS\_HARD\_DEADLOCK，死锁的详细信息会保存到deadlockDetails\[\]中。

+   死锁检测的起点，先初始化全局变量 //死锁检测开始位置，初始化边相关的全局变量如：nCurConstraints，nPossibleConstraints ,nWaitOrders,blocking\_autovacuum\_proc

```c
	//死锁检测开始位置，初始化边相关的全局变量
	nCurConstraints = 0;
	nPossibleConstraints = 0;
	nWaitOrders = 0;

	/* Initialize to not blocked by an autovacuum worker */
	blocking_autovacuum_proc = NULL;
```

+   调用DeadLockCheckRecurse函数查找环，如果找到实边死锁，直接诶返回死锁

```c
	//查找死锁（环）
	if (DeadLockCheckRecurse(proc))
	{
		int			nSoftEdges;

		TRACE_POSTGRESQL_DEADLOCK_FOUND();

		nWaitOrders = 0;
		if (!FindLockCycle(proc, possibleConstraints, &nSoftEdges))//再次检查一下死锁是否消失，若消失表名有错
			elog(FATAL, "deadlock seems to have disappeared");

		return DS_HARD_DEADLOCK;	/* 发现实边死锁*/
	}
```

+   遍历每个等待队列，将对应锁的等待进程重新添加到锁的等待队列中，并尝试唤醒一些可唤醒的进程

```c
	for (i = 0; i < nWaitOrders; i++)//遍历每个等待队列
	{
		LOCK	   *lock = waitOrders[i].lock;
		PGPROC	  **procs = waitOrders[i].procs;
		int			nProcs = waitOrders[i].nProcs;
		PROC_QUEUE *waitQueue = &(lock->waitProcs);
		ProcQueueInit(waitQueue);//初始化等待队列
		for (j = 0; j < nProcs; j++)
		{
			SHMQueueInsertBefore(&(waitQueue->links), &(procs[j]->links));//头插法插入到队列中
			waitQueue->size++;
		}
		ProcLockWakeup(GetLocksMethodTable(lock), lock);//查看是否可以唤醒一些进程
	}
```

+   如果nWaitOrders不等于0，表明还有虚边死锁，返回DS\_SOFT\_DEADLOCK
+   如果是被vacuum进程阻塞住，返回DS\_BLOCKED\_BY\_AUTOVACUUM
+   其他情况表明无死锁，返回DS\_NO\_DEADLOCK

```c
	if (nWaitOrders > 0)//有虚边死锁
		return DS_SOFT_DEADLOCK;
	else if (blocking_autovacuum_proc != NULL)//被vacuum阻塞
		return DS_BLOCKED_BY_AUTOVACUUM;
	else
		return DS_NO_DEADLOCK;//无死锁
```

### DeadLockCheckRecurse

`DeadLockCheckRecurse`函数是一个递归过程，旨在深入探索并找出有效的执行顺序以避免死锁情况。该函数的主要目的是通过递归的方式检测系统中的死锁状况，并尝试找出一个无死锁的执行顺序。它在多进程或多线程环境中特别有用，尤其是在涉及到资源共享和锁机制的情况下。

+   **参数说明**：
    
    +   `curConstraints[]`：这是一个数组，用于保存当前递归层级正在探索的边。随着递归的深入，每发现一个新的循环（即潜在的死锁条件），就会将相应的边添加到这个数组中。
    +   `waitOrders[]`：这个数组用于记录需要调整的锁等待队列，以达到一个无死锁的状态。如果存在需要调整的队列，则通过这个数组指示出来。
+   **返回值**：
    
    +   如果函数返回`true`，这意味着经过当前递归层次的探索，发现无法找到任何解决方案来避免死锁，即系统处于或即将进入死锁状态。
    +   如果返回`false`，则意味着已经找到了一种无死锁的执行顺序或调整策略，使得所有进程或线程可以在不发生死锁的情况下继续执行。此时，`waitOrders[]`中会包含如何重新排列锁等待队列的具体指导，以实现这一目标。
+   **工作原理**：
    
    +   该函数通过递归地检查当前的资源分配和锁等待关系，识别出所有可能形成环（即死锁的前提条件）的情况。
    +   对于每一个识别到的环，函数尝试添加或调整虚边（即改变某些进程的等待顺序或优先级），以打破潜在的死锁链。
    +   这个过程持续进行，直到所有可能的死锁情况都被探索完毕，或者找到了一个有效的无死锁执行方案。  
        执行流程如下
+   调用TestConfiguration检测当前的边的有效性，会将探测到的虚边保存到curConstraints数组，并返回探测到的虚边数量。
    

```c
nEdges = TestConfiguration(proc);//测试边的有效性，返回探测到的虚边数量
```

+   如果返回的虚边数量小于0，表明有实边死锁
+   如果返回的虚边数量等于0，表明无死锁
+   如果当前探测到的nCurConstraints大于maxCurConstraints，表明超出存储限制了

```c
	if (nEdges < 0)//有实边死锁
		return true;			/* hard deadlock --- no solution */
	if (nEdges == 0)//无死锁
		return false;			/* good configuration found */
	if (nCurConstraints >= maxCurConstraints)//边数量超出限制
		return true;			/* out of room for active constraints? */
```

+   判断PossibleConstraints中是否有空间，若无空间的话，就没必要保存探测到的虚边了

```c
	if (nPossibleConstraints + nEdges + MaxBackends <= maxPossibleConstraints)//有 空间保存可能的边
	{
		nPossibleConstraints += nEdges;//其实边已经存在possibleConstraints地址后了，只不过nPossibleConstraints没更新就会忽略而已
		savedList = true;
	}
	else//无空间保存
	{
		savedList = false;
	}
```

+   将探测到的虚边保存到curConstraints，然后递归调用该函数，判断是否有死锁

```c
curConstraints[nCurConstraints] = possibleConstraints[oldPossibleConstraints + i];//将探测到的边保存到curConstraints
		nCurConstraints++;
if (!DeadLockCheckRecurse(proc))//重新检测是否有死锁
			return false;		/* found a valid solution! */
```

### TestConfiguration

测试当前配置的有效性。该函数首先会调用ExpandConstraints函数尝试对等待队列进行重排来尝试解除虚边死锁；然后继续查找是否存在环，如果存在就将虚边保存到possibleConstraints数组中。

+   定义查保存虚边的地址,即possibleConstraints数组

```c
  EDGE	   *softEdges = possibleConstraints + nPossibleConstraints;//探测到的虚边都在此地址存放
```

+   判断possibleConstraints数组是否还有空间

```c
	if (nPossibleConstraints + MaxBackends > maxPossibleConstraints)
		return -1;
```

+   尝试调整等待队列

```c
	if (!ExpandConstraints(curConstraints, nCurConstraints))//尝试调整等待队列来解除虚边死锁
		return -1;
```

+   遍历每个探测到的边，尝试根据每个边的waiter和blocker进程查找环，并将找到的环中的虚边保存到softEdges中

```c
	for (i = 0; i < nCurConstraints; i++)
	{
		if (FindLockCycle(curConstraints[i].waiter, softEdges, &nSoftEdges))//查询等待者进程是否存在环
		{
			if (nSoftEdges == 0)
				return -1;		/* hard deadlock detected */
			softFound = nSoftEdges;
		}
		if (FindLockCycle(curConstraints[i].blocker, softEdges, &nSoftEdges))//查询被等待者进程是否存在环
		{
			if (nSoftEdges == 0)
				return -1;		/* hard deadlock detected */
			softFound = nSoftEdges;
		}
	}
```

+   检测当前进程是否存在环。

```c
	if (FindLockCycle(startProc, softEdges, &nSoftEdges))//检测当前进程是否存在环  
	{
		if (nSoftEdges == 0)
			return -1;	
		softFound = nSoftEdges;
	}
```

### FindLockCycleRecurse

递归查找是否存在环，查找的原理就是：在查找环时，先检查待测进程是否在visitedProcs数组中出现过，如果没出现过，就将待测进程存入到visitedProcs数组中，如果出现过，而且是在等待队列的起始处，则表明出现了死锁的环，返回死锁信息。  
例如：

1.  待测进程在visitedProcs数组中未出现过，没有环，无死锁  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7dae8c28e2e0431589ebdbbdf68a0dd0.png#pic_center)
    
2.  待测进程在visitedProcs数组中出现过，存在环，但是不在起始处，对当前进程而言不算死锁  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7dbe058119c24858b32d298ef1e87396.png#pic_center)
    
3.  待测进程在visitedProcs数组中出现过，存在环，且在起始处，存在死锁  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a7143deea17a44699312ed3bb1b68e1e.png#pic_center)
    

+   判断是否出现环

```c
	/*
	 判断是否有环，遍历visitedProcs数组，
	 如果检查的proc在数组中出现过，且是当前的进程，表明出现了环
	 */
	for (i = 0; i < nVisitedProcs; i++)
	{
		if (visitedProcs[i] == checkProc)//进程重复出现
		{
			if (i == 0)//是待检测的进程，出现了环
			{
				nDeadlockDetails = depth;

				return true;
			}
			return false;
		}
	}
```

+   如果不存在环，将待测进程保存到visitedProcs数组中

```c
visitedProcs[nVisitedProcs++] = checkProc;//没有检测到环，将检测进程存入visitedProcs数组
```

+   如果要检查的进程处于等待状态，那么就递归检测他的等待队列

```c
	if (checkProc->links.next != NULL && checkProc->waitLock != NULL &&
		FindLockCycleRecurseMember(checkProc, checkProc, depth, softEdges,
								   nSoftEdges))
		return true;
```

+   如果待测进程是锁组中的一部分，遍历锁组的每个成员进程，检查是否存在环

```c
	 如果进程没有等待，但是是锁组的一部分，还是有可能出现等待依赖边，尽管这个进程本身没有等待。
	 */
	dlist_foreach(iter, &checkProc->lockGroupMembers)//遍历每个group成员
	{
		PGPROC	   *memberProc;

		memberProc = dlist_container(PGPROC, lockGroupLink, iter.cur);

		if (memberProc->links.next != NULL && memberProc->waitLock != NULL &&
			memberProc != checkProc &&
			FindLockCycleRecurseMember(memberProc, checkProc, depth, softEdges,
									   nSoftEdges))//递归检测是否有环
			return true;
	}
```

### FindLockCycleRecurseMember

递归检查是否存在环。

+   获取锁的进程锁表，即锁模式冲突掩码

```c
	lockMethodTable = GetLocksMethodTable(lock);//获取锁方法
	numLockModes = lockMethodTable->numLockModes;//获取锁模式数量
	conflictMask = lockMethodTable->conflictTab[checkProc->waitLockMode];//获取与等待的锁模式冲突的掩码
```

+   遍历检查锁的等待队列的每个进程，如果待测进程等待的锁与当前持有的锁模式冲突，递归调用FindLockCycleRecurse函数检查是否存在环，如果存在返回实边死锁信息。

```c
procLocks = &(lock->procLocks);//获取进程锁表

	proclock = (PROCLOCK *) SHMQueueNext(procLocks, procLocks,
										 offsetof(PROCLOCK, lockLink));

	while (proclock)//遍历检查进程的等待队列的每个进程是否有死锁
	{
		PGPROC	   *leader;
		proc = proclock->tag.myProc;
		leader = proc->lockGroupLeader == NULL ? proc : proc->lockGroupLeader;
		if (leader != checkProcLeader)//同组的不检查
		{
			for (lm = 1; lm <= numLockModes; lm++)//遍历每一个锁模式
			{
				if ((proclock->holdMask & LOCKBIT_ON(lm)) &&
					(conflictMask & LOCKBIT_ON(lm)))//如果出现锁冲突，持有的锁与当前进程要等的锁模式冲突，实边
				{
					if (FindLockCycleRecurse(proc, depth + 1,
											 softEdges, nSoftEdges))//递归检查
					{
						DEADLOCK_INFO *info = &deadlockDetails[depth];//有死锁，填充死锁相关信息

						info->locktag = lock->tag;
						info->lockmode = checkProc->waitLockMode;
						info->pid = checkProc->pid;

						return true;
					}
					if (checkProc == MyProc &&
						proc->statusFlags & PROC_IS_AUTOVACUUM)//没有死锁，但是判断是否有autovacuum进程阻塞我们
						blocking_autovacuum_proc = proc;
					break;
				}
			}
		}

		proclock = (PROCLOCK *) SHMQueueNext(procLocks, &proclock->lockLink,
											 offsetof(PROCLOCK, lockLink));
	}
```

+   找到当前锁的等待队列waitOrders\[i\]，然后遍历等待队列的每个进程，如果遍历的进程等待的锁模式与待测进程锁模式冲突，递归调用FindLockCycleRecurse函数，检查是否存在环，如果存在，则为虚边死锁，将虚边保存到possibleConstraints数组中

```c
for (i = 0; i < nWaitOrders; i++)//从等待队列中找到当前锁所在的等待队列
	{
		if (waitOrders[i].lock == lock)
			break;
	}

	if (i < nWaitOrders)//判断是否找到对应的等待队列
	{
		PGPROC	  **procs = waitOrders[i].procs;

		queue_size = waitOrders[i].nProcs;

		for (i = 0; i < queue_size; i++)//遍历等待队列中的每个进程
		{
			PGPROC	   *leader;

			proc = procs[i];
			leader = proc->lockGroupLeader == NULL ? proc :
				proc->lockGroupLeader;
			if (leader == checkProcLeader) 
				break;
			if ((LOCKBIT_ON(proc->waitLockMode) & conflictMask) != 0)//等待的锁模式与待测进程的锁模式判断是否存在冲突
			{
				/* This proc soft-blocks checkProc */
				if (FindLockCycleRecurse(proc, depth + 1,
										 softEdges, nSoftEdges))//递归检查是否存在虚边环
				{
					/* fill deadlockDetails[] */
					DEADLOCK_INFO *info = &deadlockDetails[depth];//记录虚边死锁信息

					info->locktag = lock->tag;
					info->lockmode = checkProc->waitLockMode;
					info->pid = checkProc->pid;

					/*
					 即添加到possibleConstraints数组
					 */
					Assert(*nSoftEdges < MaxBackends);
					softEdges[*nSoftEdges].waiter = checkProcLeader;//保存虚边信息到即添加到possibleConstraints数组数组
					softEdges[*nSoftEdges].blocker = leader;
					softEdges[*nSoftEdges].lock = lock;
					(*nSoftEdges)++;
					return true;
				}
			}
		}
	}
```

+   如果waitOrders\[i\]中不存在当前锁的等待队列，找到该锁组的最后一个进程，检查他的等待队列的每个进程，如果遍历的进程等待的锁模式与待测进程锁模式冲突，调用FindLockCycleRecurse函数，检查是否存在环，如果存在，则为虚边死锁，将虚边保存到possibleConstraints数组中

```c
else//等待队列中没找到当前进程
	{
		PGPROC	   *lastGroupMember = NULL;
		waitQueue = &(lock->waitProcs);
		 //查找锁组的最后一个成员
		if (checkProc->lockGroupLeader == NULL)
			lastGroupMember = checkProc;
		else
		{
			proc = (PGPROC *) waitQueue->links.next;
			queue_size = waitQueue->size;
			while (queue_size-- > 0)
			{
				if (proc->lockGroupLeader == checkProcLeader)
					lastGroupMember = proc;
				proc = (PGPROC *) proc->links.next;
			}
		}
		queue_size = waitQueue->size;
		proc = (PGPROC *) waitQueue->links.next;
		while (queue_size-- > 0)//遍历等待队列，查找虚边冲突
		{
			PGPROC	   *leader;

			leader = proc->lockGroupLeader == NULL ? proc :
				proc->lockGroupLeader;

			/* Done when we reach the target proc */
			if (proc == lastGroupMember)
				break;
			if ((LOCKBIT_ON(proc->waitLockMode) & conflictMask) != 0 &&
				leader != checkProcLeader)//锁冲突
			{
				if (FindLockCycleRecurse(proc, depth + 1,
										 softEdges, nSoftEdges))//检查是否有虚边锁
				{
					DEADLOCK_INFO *info = &deadlockDetails[depth];//报错虚边死锁信息

					info->locktag = lock->tag;
					info->lockmode = checkProc->waitLockMode;
					info->pid = checkProc->pid;
					softEdges[*nSoftEdges].waiter = checkProcLeader;//保存虚边信息到即添加到possibleConstraints数组数组
					softEdges[*nSoftEdges].blocker = leader;
					softEdges[*nSoftEdges].lock = lock;
					(*nSoftEdges)++;
					return true;
				}
			}

			proc = (PGPROC *) proc->links.next;
		}
	}

```

+   无死锁

### ExpandConstraints

将边CurConstraints扩展为对受影响等待队列的新排序  
即将CurConstraints中的每个边加入到等待队列中，然后对等待队列进行拓扑排序

```c
	for (i = nConstraints; --i >= 0;)//遍历每个虚边死锁的边
	{
		LOCK	   *lock = constraints[i].lock;
		for (j = nWaitOrders; --j >= 0;)//确认等待队列中是否已经有了它
		{
			if (waitOrders[j].lock == lock)
				break;
		}
		if (j >= 0)//没遍历完，继续
			continue;
		//存入等待队列中
		waitOrders[nWaitOrders].lock = lock;
		waitOrders[nWaitOrders].procs = waitOrderProcs + nWaitOrderProcs;
		waitOrders[nWaitOrders].nProcs = lock->waitProcs.size;
		nWaitOrderProcs += lock->waitProcs.size;
		if (!TopoSort(lock, constraints, i + 1,
					  waitOrders[nWaitOrders].procs))//进行拓扑排序
			return false;
		nWaitOrders++;
	}
```

### TopoSort

当存在虚边环时，会对每个虚边调用该函数对等待队列进行重新排序，从而尝试解开虚边环。

+   将要处理的Lock的等待队列存入topoProcs数组中

```c
	proc = (PGPROC *) waitQueue->links.next;
	for (i = 0; i < queue_size; i++)//将锁的等待队列中的每个进程都保存到topoProcs数组中
	{
		topoProcs[i] = proc;
		proc = (PGPROC *) proc->links.next;
	}
```

+   初始化beforeConstraints和afterConstraints，这两个数组与topoProcs长度相等，下面要用

```c
	MemSet(beforeConstraints, 0, queue_size * sizeof(int));//初始化这俩变量，下面要用
	MemSet(afterConstraints, 0, queue_size * sizeof(int));
```

+   遍历每一个虚边
    +   判断虚边的waiter或他的groupleader是否在等待队列中，如果在里面，根据其在等待队列中的位置将beforeConstraints对应位置值+1；如果是锁组的成员，对应值置为-1；如果不在里面，跳过当前虚边，继续下一循环
    +   如果虚边的blocker或他的groupleader是否在等待队列中，如果在里面，根据其在等待队列中的位置将afterConstraints对应位置值置为其下标值+1
    +   将该虚边的pred的值置为waiter在等待队列中的位置下标，link置为afterConstraints对应位置的值。

```c
for (i = 0; i < nConstraints; i++)//遍历每一个边
	{
		proc = constraints[i].waiter;
		Assert(proc != NULL);
		jj = -1;
		for (j = queue_size; --j >= 0;)//遍历等待队列中的每个进程，与虚边的waiter对比
		{
			PGPROC	   *waiter = topoProcs[j];

			if (waiter == proc || waiter->lockGroupLeader == proc)
			{//如果waiter等于等待队列中的某个进程或他的组长，表示是一个该等待队列相关的边
				Assert(waiter->waitLock == lock);
				if (jj == -1)//是第一个，将jj标记为该进程在等待队列中的位置
					jj = j;
				else//其他的锁组成员标记为-1
				{
					Assert(beforeConstraints[j] <= 0);
					beforeConstraints[j] = -1;
				}
			}
		}
		if (jj < 0)//没有相关的等待者，表名与当前锁无关
			continue;

		/*
		 同理，判断被等待者进程
		 */
		proc = constraints[i].blocker;
		Assert(proc != NULL);
		kk = -1;
		for (k = queue_size; --k >= 0;)//遍历等待队列中的每个进程，与虚边的blocker对比
		{
			PGPROC	   *blocker = topoProcs[k];

			if (blocker == proc || blocker->lockGroupLeader == proc)
			{//如果blocker等于等待队列中的某个进程或他的组长，表示是一个该等待队列相关的边
				Assert(blocker->waitLock == lock);
				if (kk == -1)//是第一个，将kk标记为该进程在等待队列中的位置
					kk = k;
				else//其他的锁组成员标记为-1
				{
					Assert(beforeConstraints[k] <= 0);
					beforeConstraints[k] = -1;
				}
			}
		}
		if (kk < 0)//没有匹配的边，说明与当前锁无关
			continue;

		Assert(beforeConstraints[jj] >= 0);
		beforeConstraints[jj]++;	/* 如果waiter进程在topoProcs中，这里就+1*/
		constraints[i].pred = jj;//保存虚边的water在等待队列中的位置
		constraints[i].link = afterConstraints[kk];//指向afterConstraints
		afterConstraints[kk] = i + 1;
	}
```

+   开始进行拓扑排序，遍历每个等待队列中的进程
    +   从等待队列的尾部开始遍历，找到第一个非空的进程
    +   如果进程非空且其在beforeConstraints对应位置的值为0时，满足排序要求，将进程存入到waiterOrder队列中，然后将topoProcs中的对应位置置为空。
    +   更新beforeConstraints中的值。

```c
last = queue_size - 1;
	for (i = queue_size - 1; i >= 0;)//遍历topoProcs数组，反向遍历
	{
		int			c;
		int			nmatches = 0;
		while (topoProcs[last] == NULL)//找到第一个非空的进程
			last--;
		for (j = last; j >= 0; j--)
		{
			if (topoProcs[j] != NULL && beforeConstraints[j] == 0)//第一个不在虚边的
				break;
		}
		if (j < 0)//遍历完也没有不在虚边的
			return false;
		proc = topoProcs[j];
		if (proc->lockGroupLeader != NULL)//如果是锁组，获取其组长
			proc = proc->lockGroupLeader;
		Assert(proc != NULL);
		for (c = 0; c <= last; ++c)
		{
			if (topoProcs[c] == proc || (topoProcs[c] != NULL &&
										 topoProcs[c]->lockGroupLeader == proc))//当前进程或当前组长进程
			{
				ordering[i - nmatches] = topoProcs[c];//存入ordering数组即waitOrders[xx].procs
				topoProcs[c] = NULL;
				++nmatches;
			}
		}
		Assert(nmatches > 0);
		i -= nmatches;
		for (k = afterConstraints[j]; k > 0; k = constraints[k - 1].link)
			beforeConstraints[constraints[k - 1].pred]--;
	}

```

1.  查找环时找到一个虚边环，虚边为D->A, 虚边保存在CurContraints  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/70fb536e09e24f239db11f9cc0c7bdeb.png#pic_center)
    
2.  针对该虚边，遍历waiterOrders数组中的每个等待队列，并进行拓扑排序，这里假设有等待队列如下图  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3734fae3e86b4a469014482f81ef0bd5.png#pic_center)
    
3.  遍历等待队列中的每个进程并与边进行比较，将符合要求的信息存入beforeConstraints和afterConstraints，并更新curContraints的pred和link字段  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e5d4725158454a32962ac9b22eb6fcf6.png#pic_center)
    
4.  然后对等待队列进行排序，排序后的队列为  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/96caa728d87648dc851658c8f9babc65.png#pic_center)
    
5.  这样警告拓扑排序后的等待关系就变成了，下图，环被消除了，从而解决了死锁。  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bdf27d7e013f44ac95405ab8518cd90a.png#pic_center)
    

### 死锁检测示例

下面是根据上面的例子，得到的函数运行流程  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/99001e18554f4893baae67bae9728303.png#pic_center)

+   进程：A，B， C
+   锁Lock1,Lock2
+   Lock1锁等待关系： C->A->B
+   Lock2锁等待关系: B ->C
+   假设当前进程是A触发的死锁检测。  
    死锁检测流程如下：

```
* DeadLockCheck
** DeadLockCheckRecurse
*** TestConfiguration(A)
**** FindLockCycle（A，NULL，0）
***** nVisitedProcs = 0;
***** nDeadlockDetails = 0;
****** possibleConstraints = 0;
***** FindLockCycleRecurse（A，0，NULL，0）
****** visitedProcs[0] = A 
****** FindLockCycleRecurseMember(A ,A ,0,NULL,0) //事务A等待的Lock1锁的等待队列
****** Lock1的等待队列是：C->A
******* 无实边冲突
******* nWaitOrders=0
******* 获取Lock1的等待队列：A<-C 
******* FindLockCycleRecurse（C，1，NULL，0）
******* visitedProcs[0] = A
******* visitedProcs[0] = C
******* FindLockCycleRecurseMember(C ,C ,1,NULL,0) //事务C等待的Lock1锁的等待队列
******* Lock1的等待队列是：C->A
******** 无实边冲突
******** nWaitOrders=0
******** 获取Lock1的等待队列：A<-C
******** FindLockCycleRecurse（A，2，NULL，0）
********* visitedProcs[0] = A
********* visitedProcs[0] = C	
********* 在visitedProcs出现过且为起始位置，返回死锁，ndeadlockDetails=2, return true
******** deadlockDetails[0]记录虚边死锁信息 
******** possibleConstraints[0]：waiter=C , blocker=A, lock=lock1 ;nPossibleConstraints=1
******** return true 
******** return true 
******* return true 
****** return true 
***** return true 
**** return 1
*** nPossibleConstraints = 1
*** curConstraints[0]=possibleConstraints[0]=waiter=C , blocker=A, lock=lock1 
*** nCurConstraints++
*** DeadLockCheckRecurse(A) 
**** TestConfiguration(A)
***** ExpandConstraints(curConstraints,1)
****** nWaitOrders
****** waitOrders[0]=lock=lock1, procs=waitOrderProcs + nWaitOrderProcs ,nprocs=2
****** nWaitOrderProcs += 2 =2
****** TopoSort(lock1,curConstraints,2,waitOrderProcs)
******* topoProcs[0] = A 
******* topoProcs[1] = C
******* beforeConstraints[0] = 0
******* beforeConstraints[1] = 1
******* afterConstraints[0] = 1
******* afterConstraints[1] = 0
******* curConstraints[0].pred = 1
******* curConstraints[0].link = 0
******* 开始拓扑排序
******* 第一次找到进程A
******* waitOrderProcs[1] = A
******* 第二次找到进程C
******* waitOrderProcs[0] = C
******* 最终虚边顺序调整
******* return true 
****** nWaitOrders++
****** return true
***** FindLockCycle(curConstraints[i].waiter=C, softEdges, &nSoftEdges=1)
****** nVisitedProcs = 0;
****** nDeadlockDetails = 0;
****** possibleConstraints = 0;
****** FindLockCycleRecurse（C，0，NULL，0）
******* visitedProcs[0] = C 
******* FindLockCycleRecurseMember(C ,C ,0,NULL,0) //事务A等待的Lock1锁的等待队列
******** Lock1的等待队列是：C->A				
******** 无实边冲突
******** nWaitOrders=1	
******** FindLockCycleRecurse（A，1，NULL，0）
********* visitedProcs[0] = C 
********* visitedProcs[1] = A
********* FindLockCycleRecurseMember(A ,A ,0,NULL,0) //事务A等待的Lock1锁的等待队列
********** Lock1的等待队列是：C->A				
********** 无实边冲突
********** nWaitOrders=1	
********** break;
********** return false
********* return false 
******** return false 
******* return false 
****** return false 
***** FindLockCycle(curConstraints[i].blocker=A, softEdges, &nSoftEdges=1)
***** FindLockCycle(A, softEdges, &nSoftEdges=1))
****** FindLockCycleRecurse（A，1，NULL，0）
******* visitedProcs[0] = A
******* FindLockCycleRecurseMember(A ,A ,0,NULL,0) //事务A等待的Lock1锁的等待队列
******** Lock1的等待队列是：C->A				
******** 无实边冲突
******** nWaitOrders=1	
******** break;
******** return false
******* return false 	
****** return false
***** return 0
**** return false 无死锁
*** return false
** nWaitOrders=1
** 根据排序后的waitOrders，重排Lock1的等待队列为A->C
** 唤醒可以被唤醒的进程。
** return DS_SOFT_DEADLOCK 有虚边死锁 
* done
```

【参考】

1.  《PostgreSQL数据库内核分析》
2.  《Postgresql技术内幕-事务处理深度探索》
3.  《PostgreSQL指南：内幕探索》
4.  [pg14源码](https://github.com/postgres/postgres/tree/REL_14_5)