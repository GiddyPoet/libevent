# 6.4 检查事件状态

有时候需要了解事件是否已经添加,检查事件代表什么。


```cpp
int event_pending(const struct event *ev, short what, struct timeval *tv_out);

#define event_get_signal(ev) /* ... */
evutil_socket_t event_get_fd(const struct event *ev);
struct event_base *event_get_base(const struct event *ev);
short event_get_events(const struct event *ev);
event_callback_fn event_get_callback(const struct event *ev);
void *event_get_callback_arg(const struct event *ev);
int event_get_priority(const struct event *ev);

void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
```


event_pending()函数确定给定的事件是否是未决的或者激活的。如果是,而且 what 参 数设置了 EV_READ、EV_WRITE、EV_SIGNAL 或者 EV_TIMEOUT 等标志,则函数会返回事件当前为之未决或者激活的所有标志 。如果提供了 tv_out 参数,并且 what 参数中设置了 EV_TIMEOUT 标志,而事件当前正因超时事件而未决或者激活,则 tv_out 会返回事件 的超时值。



event_get_fd()和 event_get_signal()返回为事件配置的文件描述符或者信号值。 event_get_base()返回为事件配置的 event_base。event_get_events()返回事件的标志(EV_READ、EV_WRITE 等)。event_get_callback()和 event_get_callback_arg() 返回事件的回调函数及其参数指针。


event_get_assignment()复制所有为事件分配的字段到提供的指针中。任何为 NULL 的参数会被忽略。

###实例

```cpp
#include <event2/event.h>
#include <stdio.h>

/* Change the callback and callback_arg of 'ev', which must not be
 * pending. */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,
                            NULL);
    if (pending) {
        /* We want to catchthis  here so that we do not re-assign a
         * pending event.  That would be very very bad. */
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);

    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
```


----


```c
int event_pending(const struct event *ev, short what, struct timeval *tv_out);
```
该函数判断该事件是否处于pengding状态或active状态。pengding就是挂起状态，同时可以判断事件是否被删除`event_del`

* `const struct event *ev`
  * 事件 
* `short what`
  * 返回事件类型
* `struct timeval`
  * 如果是EV_TIMEOUT事件，如果tv_out非空，则会返回时间


> 补充下未决非未决，和激活状态的概念
> 当事件被`event_new`创建后进入非未决的状态
> 当事件被`event_add`加入后，变成未决状态
> 当进入时间循环，同时对应的事件被触发，从未决状态变为激活态
> 通过`event_del`删除事件，将未决态变为非未决态

Events have similar lifecycles. Once you call a Libevent function to set up an event and associate it with an event base, it becomes initialized. At this point, you can add, which makes it pending in the base. When the event is pending, if the conditions that would trigger an event occur (e.g., its file descriptor changes state or its timeout expires), the event becomes active, and its (user-provided) callback function is run. If the event is configured persistent, it remains pending. If it is not persistent, it stops being pending when its callback runs. You can make a pending event non-pending by deleting it, and you can add a non-pending event to make it pending again.



> 可以通过`event_self_cbarg()`将`event_new`将自己生成的event作为参数传递给回调函数


### 通过event_assign修改回调函数

```c
event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);

event_assign(ev, base, fd, events, new_callback, new_callback_arg);
```

