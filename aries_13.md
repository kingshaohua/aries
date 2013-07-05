**1.3 总结**
本文中，我们阐释了ARIES恢复策略，并且说明了为什么System R中的一些恢复模型不适用于WAL场景。我们讨论了一些特性，这些特性对于构建和操作工业尺度的事务系统很重要。这里讨论了一些议题：操作日志，低颗粒度锁，空间管理，灵活恢复。简单的说：ARIES可以达到我们所设定的目标，通过记录每个页的更新操作，用每个页上的LSN来跟踪页状态，重启恢复时在undo失效事务之前进行历史重演，并且将CLR和他们要补偿的日志记录串连起来。对ARIES的使用并不局限于数据库领域。它也可以用来实现持久化的面向对象语言，文件恢复系统，基于事务的操作系统。实际上，它已经用于QuickSilver分布式操作系统【40】上了，并在某个系统中设计用来向某个主机添加工作区数据备份功能。  
在本章节中，我们总结ARIES的一些特性，可以使我们的系统更加高效和灵活。  
历史重演，实际上使用的是LSN和在undo时写CLR,不管有没有使用UndoNxtLSN字段来串连CLR，都可以允许以下的特性：  
（1）支持记录锁，并且支持在页内移动记录（防止碎片），而不需要对记录加锁，也不需要写“移动”日志。  
（2）只使用一个状态变量，每页一个日志序列号。  
（3）对存储的重复利用：一个事务内的重复利用，或者不同事务间的重复利用。这样保留了数据的聚集性，也提高了存储的利用率。  
（4）正常流程中的原始操作的逆序和在对其undo是的顺序不同（比如：对空间映射表的变更）。也就是说，这是使用逻辑undo进行恢复独立性称为可能。  
（5）多个事务在执行时，可以并发undo同一个页。  
（6）页的恢复独立于其他页或者事务相关的日志记录，尤其是在介质恢复的时候。  
（7）如果必要，在系统崩溃时的事务，在恢复后可以继续执行。  
（8）选择或者延迟重启，以及在undo失效事务时并行开启新事务，提高了数据可用性。  
（9）事务的部分回滚。  
（10）对一个页记录操作日志和逻辑日志。例如：对于自增自减记录逻辑日志，而不是记录数据修改前后的数据镜像。  
使用UndoNxtLSn来串连CLR运行以下一些特性（同样要提供历史重演）：  
（1）废弃undo CLR，这样就避免对CLR还要写CLR。这样也没必要在CLR中存储undo信息。  
（2）在正常处理时，对于同一条记录的undo至多写一次。  
（3）当一个事务正在回滚时，当某对象被undo了，该对象上的锁就可以释放了。在回滚耗时较长的事务或者，用部分回滚解决死锁问题的时候，就尤为重要了。
（4）不需要用特殊流程处理部分回滚，比如像System R中打补丁日志。  
（5）如果需要的话，使用内嵌顶级任务来执行持久化，对于那些不管事务最后是提交还是回滚必须落地的变更。  
在历史重演之前执行分析遍历，可以允许一下特性：  
（1）在恢复的redo,undo遍历的任意时刻执行checkpoint。  
（2）动态的将文件返回给操作系统，因此就允许数据库对象和文件直接的动态管理。  
（3）恢复文件相关信息时可以并行的恢复用户数据，而不需要对前者做特殊处理。  
（4）标记那些可能会redo的页，这样在redo遍历前就能并行异步加载它们。  
（5）通过消除来自于脏页表的一些页，来减少一些redo工作。比如一些空页已经被释放了。  
（6）在redo时，避免读取一些页。比如在脏页落地时输出end_write记录，如果遇到end_write记录就从脏页表中去除这些页。  
（7）标识那些in-doubt和in-progress态的事务。这样在redo遍历的时候就能持有相应的锁，从而可以支持选择或者延迟重启，在重启之后也能继续失效事务，并且在undo那些失效事务的同时，还可以开启新事务。  
**13.1 Implementations and Extensions 实现与扩展**  
ARIES提出了恢复算法的原理，运用在了：IBM搜索原型系统Starburst【87】和 QuickSilver【40】，Wisconsin 大学的EXODUS 和 Gamma 数据库系统【20】，IBM产品0S/2 Extended Edition Database Manager【7】以及WorkstationData Save Facility/VM【44】。ARIES的一种特性"历史重演"，运用在了DB2 Version 2 Release 1，并使用了内嵌顶级任务来支持段式表空间。【98】中有一个对于ARIES性能的模拟研究。该课题称“模拟结果表明，ARIES恢复策略在提供快速恢复（由长时间的内部checkpoint导致的系统崩溃），对页LSN，日志LSN，RecLSN的高效使用，避免了不必要的redo更新，并且可以减轻对实际恢复的压力。除此之外，由事务的并发控制和恢复算法所导致的额外负载很低，这在事务响应时间和事务平均执行时间上相差很小（如果是运行在一个没有崩溃的系统上）。该研究同样结合了一些证据，该恢复方法在通过低颗粒度进行并发控制方面处理的很好，这是一个很重要的优点。”  
我们已经扩展了ARIES，使其能够工作在内嵌式事务模型下（参见【70,85】）。基于ARIES，我们开发了一些新方法：ARIES /KVL, ARIES/IM 和 ARIES /LHS，可以高效的操作和恢复B+树索引【57,62】和hash存储数据结构【59】。同样，我们还扩展了ARIES来限制对失效事务执行历史重演的数量【69】。基于ARIES，我们设计未N-way数据共享（比如磁盘共享）环境【65,66,67,68】设计了并行控制和恢复算法。Commit_LSN,这利用了page_LSN 的优势，保存在每个页中来减少加锁，加闩，重新评估的负载，并且提高了并行性，这在【54,58,60】中有详细描述。尽管消息也是事务系统的一个重要组成部分，在本文中，我们不会讨论消息的日志记录和恢复。  
**ACKNOWLEDGMENTS（致谢，这就不翻译了~）**  
We have benefited immensely from the work that was performed in the System R project and in the DB2 and IMS product groups. We have learned valuable lessons by looking at the experiences with those systems. Access to the source code and internal documents of those systems was very helpful. The Starburst project gave us the opportunity to begin from scratch and design some of the fundamental algorithms of a transaction system, taking into account experiences with the prior systems. We would like to acknowledge the contributions of the designers of the other systems. We would also like to thank our colleagues in the research and product groups that have adopted our research results. Our thanks also go to Klaus Kuespert, Brian Oki, Erhard Rahm, Andreas Reuter, Pat Selinger, Dennis Shasha, and Irv Traiger for their detailed comments on the paper.
