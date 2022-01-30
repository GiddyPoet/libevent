# 5.1 运行循环

一旦有了一个已经注册了某些事件的 event_base(关于如何创建和注册事件请看下一节 ), 就需要让 libevent 等待事件并且通知事件的发生。

```cpp
// 设置为EVLOOP_ONCE，那么event_loop就会等待到第一个事件超时，处理在这段时间内激活的event，直到所有激活的事件都处理完就退出event_loop
#define EVLOOP_ONCE             0x01
// 设置为EVLOOP_NONBLOCK，那么event_loop只会处理当前已经激活的event，处理结束后就会退出event_loop
#define EVLOOP_NONBLOCK         0x02
// 不退出，直到调用event_base_loopbreak()或者 event_base_loopexit()为止。
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04

int event_base_loop(struct event_base *base, int flags);
```

默认情况下,event_base_loop()函数运行 event_base 直到其中没有已经注册的事件为止。执行循环的时候 ,函数重复地检查是否有任何已经注册的事件被触发 (比如说,读事件 的文件描述符已经就绪,可以读取了;或者超时事件的超时时间即将到达 )。如果有事件被触发,函数标记被触发的事件为 “激活的”,并且执行这些事件。


在 flags 参数中设置一个或者多个标志就可以改变 event_base_loop()的行为。如果设置了 EVLOOP_ONCE ,循环将等待某些事件成为激活的 ,执行激活的事件直到没有更多的事件可以执行,然会返回。如果设置了 EVLOOP_NONBLOCK,循环不会等待事件被触发: 循环将仅仅检测是否有事件已经就绪,可以立即触发,如果有,则执行事件的回调。

完成工作后,如果正常退出, event_base_loop()返回0;如果因为后端中的某些未处理 错误而退出,则返回 -1。


为帮助理解,这里给出 event_base_loop()的算法概要:

```cpp
while (any events are registered with the loop,
        or EVLOOP_NO_EXIT_ON_EMPTY was set) {

    if (EVLOOP_NONBLOCK was set, or any events are already active)
        If any registered events have triggered, mark them active.
    else
        Wait until at least one event has triggered, and mark it active.

    // 根据优先级列表执行
    for (p = 0; p < n_priorities; ++p) {
       if (any event with priority of p is active) {
          Run all active events with priority of p.
          break; /* Do not run any events of a less important priority */
       }
    }

    if (EVLOOP_ONCE was set or EVLOOP_NONBLOCK was set)
       break;
}
```

为方便起见,也可以调用

```cpp
int event_base_dispatch(struct event_base *base);
```

event_base_dispatch ()等同于没有设置标志的 event_base_loop ( )。所以, event_base_dispatch ()将一直运行,直到没有已经注册的事件了,或者调用 了 event_base_loopbreak()或者 event_base_loopexit()为止。


这些函数定义在<event2/event.h>中,从 libevent 1.0版就存在了。


```c
int
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;
	struct timeval tv;
	struct timeval *tv_p;
	int res, done, retval = 0;

	/* Grab the lock.  We will release it inside evsel.dispatch, and again
	 * as we invoke user callbacks. */
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	if (base->running_loop) {
		event_warnx("%s: reentrant invocation.  Only one event_base_loop"
		    " can run on each event_base at once.", __func__);
		EVBASE_RELEASE_LOCK(base, th_base_lock);
		return -1;
	}

	base->running_loop = 1;

	clear_time_cache(base);

	if (base->sig.ev_signal_added && base->sig.ev_n_signals_added)
		evsig_set_base_(base);

	done = 0;

#ifndef EVENT__DISABLE_THREAD_SUPPORT
	base->th_owner_id = EVTHREAD_GET_ID();
#endif

	base->event_gotterm = base->event_break = 0;

	while (!done) {
		base->event_continue = 0;
		base->n_deferreds_queued = 0;

		/* Terminate the loop if we have been asked to */
		if (base->event_gotterm) {
			break;
		}

		if (base->event_break) {
			break;
		}

		tv_p = &tv;
		if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {
			timeout_next(base, &tv_p);
		} else {
			/*
			 * if we have active events, we just poll new events
			 * without waiting.
			 */
			evutil_timerclear(&tv);
		}

		/* If we have no events, we just exit */
		if (0==(flags&EVLOOP_NO_EXIT_ON_EMPTY) &&
		    !event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
			event_debug(("%s: no events registered.", __func__));
			retval = 1;
			goto done;
		}

		event_queue_make_later_events_active(base);

		clear_time_cache(base);

		res = evsel->dispatch(base, tv_p);

		if (res == -1) {
			event_debug(("%s: dispatch returned unsuccessfully.",
				__func__));
			retval = -1;
			goto done;
		}

		update_time_cache(base);

		timeout_process(base);

		if (N_ACTIVE_CALLBACKS(base)) {
			int n = event_process_active(base);
			if ((flags & EVLOOP_ONCE)
			    && N_ACTIVE_CALLBACKS(base) == 0
			    && n != 0)
				done = 1;
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;
	}
	event_debug(("%s: asked to terminate loop.", __func__));

done:
	clear_time_cache(base);
	base->running_loop = 0;

	EVBASE_RELEASE_LOCK(base, th_base_lock);

	return (retval);
}
```

```c
static int
event_process_active(struct event_base *base)
{
	/* Caller must hold th_base_lock */
	struct evcallback_list *activeq = NULL;
	int i, c = 0;
	const struct timeval *endtime;
	struct timeval tv;
	const int maxcb = base->max_dispatch_callbacks;
	const int limit_after_prio = base->limit_callbacks_after_prio;
	if (base->max_dispatch_time.tv_sec >= 0) {
		update_time_cache(base);
		gettime(base, &tv);
		evutil_timeradd(&base->max_dispatch_time, &tv, &tv);
		endtime = &tv;
	} else {
		endtime = NULL;
	}

    // 遍历优先级队列，执行
	for (i = 0; i < base->nactivequeues; ++i) {
		if (TAILQ_FIRST(&base->activequeues[i]) != NULL) {
			base->event_running_priority = i;
			activeq = &base->activequeues[i];
			if (i < limit_after_prio)
				c = event_process_active_single_queue(base, activeq,
				    INT_MAX, NULL);
			else
				c = event_process_active_single_queue(base, activeq,
				    maxcb, endtime);
			if (c < 0) {
				goto done;
			} else if (c > 0)
				break; /* Processed a real event; do not
					* consider lower-priority events */
			/* If we get here, all of the events we processed
			 * were internal.  Continue. */
		}
	}

done:
	base->event_running_priority = -1;

	return c;
}
```