ARIES/IM:基于WAL的一种高效高并发的索引管理方法  
C. MOHAN
Data Base Technology Institute, IBM Almaden Research Center, San Jose, CA 95120, USA
wharr@alnwden, tbm. com
FRANK LEVINE
IBM, 11400 Burnet Road, Austin, TX 78758, USA  
摘要：本文提供了一种对于事务系统中索引管理的综合处理方案。这种方案叫做ARIES/IM(Algorlthm for Recovery and isolation Exploiting Semantics for Index Management),可以进行并发控制和B+树的恢复。ARIES/IM确保串行性并使用WAL来恢复。它支持高并发并且有性能优异：（1）将某个key上的锁视为数据页中相应记录的锁（比如在记录级别）（2）为了支持高并发，在提交时不获取索引页锁，即使这时候索引结构发生了变更（SMO）比如页分裂，页删除。（3）支持在遍历，插入，删除同时，进行SMO。在重启恢复时，对于索引变更的redo是以面向页的方式执行（比如，不需要遍历整个索引树），并且在正常执行和重启恢复时，不管有没有可能，undo都是以面向页的方式执行的。ARIES/IM使用多种级别的锁来保证灵活性。ARIES/IM的一个子集已经应用到0S/2 Extended Edition Database Manager。由于ARIES/IM的锁设计很通用，所以它们也被应用到SQL/DS和VM Shared File System，尽管这些系统使用影子页技术来恢复。  
**1、介绍**  
对B树及其变种的并发访问协议已经研究了好长时间(参见 [BaSc77, LeYa81, Mino84, Sagi86, ShGo88]及其它们所引用的一些文章）。这些论文中都没有考虑的如何保证事务的原子性和串行性，该事务包含了对B+树的多种操作（比如获取，插入，删除等）,当事务，系统，或者介质崩溃，并且被不同事务同时访问。[FuKa89]描述了一种错误的（比如：在not found的时候不完全锁，对于范围扫描时进行完全加锁）并且昂贵（使用嵌套事务）的方案来解决此类问题（详情参考[MoLe89]）。数据库管理系统（DBMS）比如[DB21,the 0S12 Extended Edition Database Managerl, System R,NonStop SQLt and SQUDS]的索引管理都支持串行化（重复读（RR）或者第三级别的一致性[Gray78]）。在恢复时，DB2, NonStop SQL 和  0S/2 Extended Edition Database Manager 使用日志先行（WAL）[Gray78,MHLPS92],而System R 和 SQL/DS使用影子页技术[GM BLL81]。不幸的是，上述系统所使用算法细节并没不公开。本文中，我们会描述一种并发控制及恢复策略，称为ARIES/IM（A/gorithm for Recovery and Isolation Exploiting Semantics forindex Management),来构建B+树索引。我们使用ARIES/IM作为0S/2 Extended Edition Database Manager设计的一部分。  
首先，System R如何对索引加锁的大部分细节已经在[Moha90a]中详细描述，并且作为我们ARIES/KVL的一部分，用来改进该方法的并行度和加锁开销特性。除了提供低颗粒度锁（通过数据的记录锁和索引的key值锁），System R系统（起源于IBM的SQL/DS产品）锁提供的并发级别，客户并不满意的。由于，ARIES/KVL是在key值上加锁而不是在单独的key上加锁，所以ARIES/KVL对于增强并发度仍然不够。后者在非唯一性索引上有重大改进。此外，在System R中，即使是对单条记录的插入或删除所要获取的锁数量也是相当多的。因此在设计ARIES/IM是，我们的首要目标是修改System R算法，使其使用WAL，并且彻底提升了它的并发度，性能和功能特性。需要通过高效的恢复和存储以及高并发来支持串行执行。ARIES/IM可以满足所有这些要求。  
ARIES/IM是基于ARIES的恢复和并行控制策略，这在[MH LPS92]中有介绍，并且在众多产品中不同程度的实现了它，比如：IBM的产品0S/2 Extended Edition Database Manager,Workstation Data Save FacilityA/M 和 DB2 V2,IBM的研究原型系统Starburst 和 QuickSilver，以及Transarc的Encinai产品套装，Wisconsin大学的 Gamma data base machine 和 EXODUS 可扩展DBMS。在ARIES/IM中，以极少的锁提供了高度的并发性。通过高效的执行undo和redo以及在undo时避免死锁，来提升重启恢复和正常执行时的性能。我们的并发度测量在[KuPa79]中有明确的定义，它陈述：一系列事务运行相交的事务越多，它们的并行度越高。我们的性能测量就是所需要获取的锁数量，在redo,undo过程中访问的页数量，以及正常操作数，在介质恢复时所需要遍历日志的次数，所需要同步的数据页数，以及日志IO。  
![](./img/fig1.png)  
本文剩余的部分如下组织。本节接下来，我们首先介绍树的结构，并列出在索引并发控制和恢复中的一些问题。由于ARIES/IM是基于ARIES的，我们会介绍ARIES的恢复和并发控制策略。接着第二节会描述ARIES/IM的并发控制特性，第三节讨论恢复方面。第四节，我们会解释如何去避免死锁以及为何回滚事务不会触发死锁。在第五节，我们做一个总结，并讨论ARIES/IM的实现。  
**1.1. Tree Architecture and Problems**  
一个叶子页中的key就是一组key值和record-id，record-id(RID)就是包含了这个key值的记录的标识。记录是存储在数据页中（在索引树之外）。叶子页是单独的前后串连的。每一个非叶子页都包含了一定数量的子页指针和少量的high key--每个high key都和一个子页指针关联，最右的子页就没有high key和它关联。对于某个子页，存储在非叶子页中的high key通常比存储那个子页中的最大的high key要大。  
ARIES/IM支持4中基本的索引操作：  
(1)Fetch获取：对于某个key值或者部分key值（比如前缀），检查其是否在索引中，并且取出完整的key。同样可以使用起始条件（=，>,>=）。  
(2)Fetch Next获取下一个：使用Fetch调用打开一个范围遍历，获取下一个满足条件范围的key（比如，终结key或者比较操作符（<,=,<=））。  
(3)Insert 插入：插入某个key(key-value,RID)。对于唯一性索引，会首先调用search逻辑来避免key重复。对于非唯一性索引，这个新key会作为search的输入。  
(4)Delete 删除：删除指定的key。  
在对索引树进行恢复和并发修改时有很多问题，一些接下来会被解答的问题是：  
（1）如何日志记录索引的变更，这样在系统崩溃后进行恢复时，丢失的更新都能快速的重新应用上？  
（2）如何确保，如果一个SMO(structure modification operation,比如，页分裂，或者页删除操作)正在执行，此时系统崩溃了，该SMO的一部分操作影响已经记录到数据库的磁盘上，那么系统可以将索引树恢复到系统奔溃时刻的结构，并保证其一致性（参见图11，该情形会导致树结构的不一致）。  
（3）怎样对索引页执行变更，才能最小程度的减少对其他访问者的影响。  
（4）如何确保即使某个事务在成功提交了一次SMO后回滚了，它不会undo这个SMO，因为这么做会导致丢失其他事务的一些更新操作（其他事务在此期间对于这个SMO涉及的页进行的操作）  
（5）如何检测事务T1原来插入到页P1的key已经被移到了P2上（因为事务T2的SMO操作），这样，如果T1回滚了，然后访问到了P1，发现P1上的key已经被删了。（参见图1中的逻辑undo的例子）  
（6）如何检测如果事务T1从P1删除的key，由于其他事务的SMO操作，该key又处于P2上，这样当T1回滚的时候，就会访问到P2上，但是key就会已存在。  
（7）如何避免回滚事务的死锁，这样对于回滚事务的时候就不需要特别的流程来避免死锁。  
（8）如何支持多种颗粒度级别锁，选择什么作为锁对象。  
（9）如何锁住“not found”状态来确保RR（比如，处理幻读问题。）  
（10）如何保证在唯一性索引上，如果一个事务删除了一个key值，在这个事务提交前，其他事务不可以使用该key值。  
（11）如何使的树的遍历能够持续进行，即使正在执行SMO，并且能够确保该遍历的事务可以恢复，(这些事务涉及到了SMO影响的页)参见图3，描述了该场景。  
**1.2ARIES**  
在本一小节中，我们会简略的介绍一下ARIES恢复策略。读者可以参考[MHLPS92]，其中有对ARIES的详细描述，[MoPi91]对ARIES的实现方法做了一些优化，[MoNa91]正对ARIES在共享磁盘环境下做了增强，[RoM089]描述了ARIES/NT,这是对嵌套事务模型的ARIES的扩展。我们假设读者对以下概念已经有所了解：latch，不同级别的一致性（重复读，游标稳定性），不同阶段（tnstant,提交）以及不同的latch,锁模型，这些在[Gray78, MHLPS92, Moha90a, Moha90b]中有详细描述。  
在ARIES中，每个数据页都有一个page_LSN字段包含日志记录的LSN，用它来描述最近一次对页做的更新。由于LSN随着时间递增，通过比较恢复时刻的page_LSN和日志记录的LSN，我们就可以确定该条日志是否已经在该页上生效。也就是说，如果page_LSN比日志记录的LSN小，那么后者就还没有在该数据页上生效。ARIES在页上使用latch来保证相关信息的物理一致性，在数据上用锁来保证逻辑一致性。ARIES支持低颗粒度锁（比如记录锁），以及富语义的锁模型（比如，自增/自减锁），部分回滚，嵌套事务，日志先行，选择和延迟重启，模糊进行备份（日志归档），介质恢复，强制和非强制缓存管理策略。  
在ARIES中，系统崩溃后重启恢复包含三次日志遍历：analysis,redo,undo。第一次日志扫描，从上一次完成的checkpoint开始到日志结束。analysis遍历决定下一次日志扫描的起始点。它同样会提供in-flight和in-doubt状态的事务列表。在redo遍历时，ARIES使用日志重演来redo那些磁盘上的日志更新（这些更新之前已经更新到数据页上但是没有落地到磁盘中）。对于所有事务都这么处理，包括那些in-flight的事务。redo遍历同样会获取一些锁来保护那些in-doubt事务中未提交的更新。  
接下来的遍历是undo遍历，这时所有in-flight的事务的更新都会回滚（在单一日志文件范围内，以逆序方式回滚）。除了在事务正常执行时记录日志，ARIES同样也会记录（通常使用补偿日志，CLR）那些在部分或者回滚事务中的更新操作。CLR有一个属性：他们都是redo-only日志。通过将CLR和正常执行的日志记录串连起来，回滚所需的日志量就固定了（即使在重启恢复时重复崩溃或者嵌套回滚）。当对一条日志（nonCLR）进行undo时，会生成一条CLR，（通过CLR的UndoNxtLSN字段）该CLR会指向这条undo日志的前继。  
有时候我们希望事务中的一些更新肯定要被提交，不管该事务有没有提交。我们仍需要这些操作的原子性。对于某些情况下很实用：本文接下来会提到的索引页的分裂和删除，以及在hash存储模型中的记录重定向[Moha92]。ARIES通过内嵌顶级动作来支持这一特性。预期的效果是在内嵌顶级动作结束时生成虚拟CLR（参见图9）。虚拟CLR的UndoNxtLSN记录了开始内嵌顶级动作前一条日志记录，如果事务在完成内嵌顶级动作之后准备回滚。ARIES的历史重演特性保证了内嵌顶级动作可以redo，如果有需要的话，在系统崩溃后，即使他们已经在in-flight事务中生效了。如果系统在生成虚拟CLR之前崩溃，那么这个不完整的内嵌顶级动作就会被undo，通过记录在undo-redo(对应于redo-only)日志记录中内嵌顶级动作的日志来undo。这提供了内嵌顶级动作本身的原子属性。  
