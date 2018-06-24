# 惊群效应
惊群效应：当多个进程或多个线程等待同一个互斥资源时，当资源可用时，等待的进程或线程会被全部唤醒，然后，只有一个进程或线程可以成功获得该互斥资源。

## linux内核的解决策略
linux内核使用一个flag将等待队列中的进程分为两种：
- flags = 1：互斥进程
- flags = 0：非互斥进程

使用两种不同的函数将进程加入到等待队列中：
- Add_to_wait_exclusive()  **将互斥进程加入到等待队列尾部**
- Add_wait_queue() **将非互斥进程加入到等待队列头部**

唤醒进程的时候：
```cpp
void wake_up(wait_queue_head_t *q) {
	srtuct list_head *tmp;
	wait_queue_t *curr;
	
	list_for_each(tmp, &q->task_list) {
		curr = list_entry(tmp, wait_queue_t, task_list);
		if (curr->func(curr, TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE, 0, NULL) && curr->flags) 
			break;
	}
}
```
**curr->flags表明如果该进程是互斥进程就直接返回了，而且linux内核还采用了一个小心机：对于非互斥进程，应该被全部唤醒，所以飞互斥进程会被加入到队列头部，而互斥进程则加入到队列尾部**

## 网络编程中常见惊群效应
- accept：新版本内核已经修复，即目前的accept不会产生惊群效应，因为如上的等待队列解决方案一样；
- epoll：以前实验室的师兄曾遇到的问题：将listenfd加入到epollfd中，多线程共享epollfd，导致在accept时候出现惊群。原因：**lt模式下，对于同一个文件描述符在epollfd中一直返回，直到该文件描述符不可读/可写，而线程取到该文件描述符时候可能还没有执行accept操作时epollfd又返回了，于是其他线程会对同一个文件描述符进行同样的操作。(epoll处于lt模式下，epoll_wait会自动将所有就绪的fd再次加入到就绪链表中)。其实epoll已经在源码层次避免了惊群效应，如等待队列一样的做法，出现惊群只是使用的方法不够合理。**

Nginx提出了一种优秀的解决方案：多个进程都在epoll_wait，但是都没有accept的权利，只有得到accept_mutex的进程才有权力进行accept操作，从而将该文件描述符加入到自己的epollfd中；