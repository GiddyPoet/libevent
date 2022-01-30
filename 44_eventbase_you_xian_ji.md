# 4.4 event_base优先级

libevent支持为事件设置多个优先级。然而, event_base默认只支持单个优先级。可以调用 event_base_priority_init()设置 event_base 的优先级数目。

```cpp
int event_base_priority_init(struct event_base *base, int n_priorities);
```

成功时这个函数返回 0,失败时返回 -1。base 是要修改的 event_base,n_priorities 是要支 持的优先级数目,这个数目至少是 1 。每个新的事件可用的优先级将从 0 (最高) 到 n_priorities-1(最低)。


常量 EVENT_MAX_PRIORITIES 表示 n_priorities 的上限。调用这个函数时为 n_priorities 给出更大的值是错误的。

>必须在任何事件激活之前调用这个函数,最好在创建 event_base 后立刻调用。


> 该函数为event_base创建优先级队列，用于基于base

```c
int
event_base_priority_init(struct event_base *base, int npriorities)
{
	int i, r;
	r = -1;

    // 加锁
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

    // 判断base和npriorities的范围
	if (N_ACTIVE_CALLBACKS(base) || npriorities < 1
	    || npriorities >= EVENT_MAX_PRIORITIES)
		goto err;

    // 已经设定过
	if (npriorities == base->nactivequeues)
		goto ok;

    // 判断队列状态
	if (base->nactivequeues) {
		mm_free(base->activequeues);
		base->nactivequeues = 0;
	}

    // 开辟优先级队列
	/* Allocate our priority queues */
	base->activequeues = (struct evcallback_list *)
	  mm_calloc(npriorities, sizeof(struct evcallback_list));
	if (base->activequeues == NULL) {
		event_warn("%s: calloc", __func__);
		goto err;
	}
	base->nactivequeues = npriorities;

    // 初始化优先级队列
	for (i = 0; i < base->nactivequeues; ++i) {
		TAILQ_INIT(&base->activequeues[i]);
	}

ok:
	r = 0;
err:
	EVBASE_RELEASE_LOCK(base, th_base_lock);
	return (r);
}
```


