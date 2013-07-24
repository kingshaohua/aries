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
（2）如何确保，如果一个SMO(structure modification operation,比如，页分裂，或者页删除操作)正在执行，此时系统崩溃了，该SMO的一部分操作影响已经记录到数据库的磁盘上，那么系统可以重新装载该树的结构一致性