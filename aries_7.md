**7. CHECKPOINTS DURING RESTART 重启时构建checkpoint**  
在本章节中，我们会讨论：如何在重启过程中的不同阶段执行checkpoint，从而可以减少对CPU和IO的负载。  
**分析遍历**  
在分析遍历结束后执行checkpoint，这样在恢复的失败的话，可以节省一部分工作。checkpoint中事务表要和分析遍历完成时的事务表一致。checkpoint的脏页列表要和分析遍历的脏页表一致。对于后者，脏页列表取自于缓存池(BP)脏页表。  
**Redo遍历**  
在redo遍历之初,将会通知缓存管理(BM)，在遍历时，当它刷出页数据到非易失存储上时，它会变更脏页表使其RecLSN为该日志记录的LSN，这样到此日志记录的所有日志都会被处理。如果BM以这种方式管理重启脏页表就足够了，BM在正常流程时不需要维护自己的脏页表。当然，它仍需要跟踪当前处于缓存中的页。在redo遍历时，可以在任意时刻执行checkpoint。如果在遍历结束前系统崩溃，可以减少下次需要redo的日志量。checkpoint的脏页列表和此时checkpoint的重启脏页表一致。该checkpoint的事务表和分析遍历结束时的事务表一致。建立checkpoint，并不会受并发redo遍历的影响。 
**Undo遍历**  
在undo遍历之初，重启脏页表变成了BP脏页表。
