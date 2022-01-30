# 4.1 创建event_base


###4.1.1 创建默认的event_base

***event_base_new()***函数分配并且返回一个新的具有默认设置的 event_base。函数会检测环境变量,返回一个到 event_base 的指针。如果发生错误,则返回 NULL。选择各种方法时,函数会选择 OS 支持的最快方法。

```cpp
struct event_base *event_base_new(void);
```

>大多数程序使用这个函数就够了。

event_base_new()函数声明在<event2/event.h>中,首次出现在 libevent 1.4.3版。


###4.1.2 创建复杂的event_base

要对取得什么类型的 event_base 有更多的控制,就需要使用 **event_config**。


event_config 是一个容纳 event_base 配置信息的不透明结构体。需要 event_base 时,将 event_config 传递给**event_base_new_with_config ()。**

###创建接口
```cpp
struct event_config *event_config_new(void);

struct event_base *
event_base_new_with_config(const struct event_config *cfg);

void event_config_free(struct event_config *cfg);
```

要使用这些函数分配 event_base,先调用 event_config_new()分配一个 event_config。 然后,对 event_config 调用其它函数,设置所需要的 event_base 特征。最后,调用 event_base_new_with_config()获取新的 event_base。完成工作后,使用 event_config_free ()释放 event_config。



```cpp
int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);

enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
```

调用 event_config_avoid_method ()可以通过名字让 libevent 避免使用特定的可用后端 。 调用 event_config_require_feature ()让 libevent 不使用不能提供所有指定特征的后端。 调用 event_config_set_flag()让 libevent 在创建 event_base 时设置一个或者多个将在下面介绍的运行时标志。

**event_config_require_features()可识别的特征值有:**

* EV_FEATURE_ET:要求支持边沿触发的后端
* EV_FEATURE_O1:要求添加、删除单个事件,或者确定哪个事件激活的操作是 O(1)复杂度的后端
* EV_FEATURE_FDS:要求支持任意文件描述符,而不仅仅是套接字的后端

**event_config_set_flag()可识别的选项值有:**

* EVENT_BASE_FLAG_NOLOCK :不要为 event_base 分配锁。设置这个选项可以 为 event_base 节省一点用于锁定和解锁的时间,但是让在多个线程中访问 event_base 成为不安全的。


* EVENT_BASE_FLAG_IGNORE_ENV :选择使用的后端时,不要检测 EVENT_* 环境 变量。使用这个标志需要三思:这会让用户更难调试你的程序与 libevent 的交互。
* EVENT_BASE_FLAG_STARTUP_IOCP:仅用于 Windows,让 libevent 在启动时就 启用任何必需的 IOCP 分发逻辑,而不是按需启用。
* EVENT_BASE_FLAG_NO_CACHE_TIME :不是在事件循环每次准备执行超时回调时 检测当前时间,而是在每次超时回调后进行检测。注意:这会消耗更多的 CPU时间。
* EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST :告诉 libevent ,如果决定使 用 epoll 后端,可以安全地使用更快的基于 changelist 的后端。epoll-changelist 后端可以 在后端的分发函数调用之间,同样的 fd 多次修改其状态的情况下,避免不必要的系统 调用。但是如果传递任何使用 dup()或者其变体克隆的 fd 给 libevent,epoll-changelist 后端会触发一个内核 bug,导致不正确的结果。在不使用 epoll 后端的情况下,这个标 志是没有效果的。也可以通过设置 
* EVENT_EPOLL_USE_CHANGELIST 环境变量来 打开 epoll-changelist 选项。
* EVENT_BASE_FLAG_PRECISE_TIMER 使用精度高的定时器如CLOCK_MONOTONIC，如果系统支持timerfd_create()则会在epoll使用此系统定时器


上述操作 event_config 的函数都在成功时返回0,失败时返回-1。

>设置 event_config,请求 OS 不能提供的后端是很容易的 。比如说,对于 libevent 2.0.1-alpha, 在 Windows 中是没有 O(1)后端的;在 Linux 中也没有同时提供 EV_FEATURE_FDS 和 EV_FEATURE_O1 特征的后端。如果创建了 libevent 不能满足的配置, event_base_new_with_config ()会返回 NULL。



----

## 删除特定的I/O机制

`int event_config_avoid_method(struct event_config *cfg, const char *method)`函数功能：
> 由于event base 支持多种io机制,如poll/epoll/select等,若我们不想让让event base选择哪个,那么我们就可以用此函数设置禁止的io机制要想禁止多个可以多次调用此函数,内部用链表保存禁止的项

```
    参数:
        cfg: 内部配置结构体指针
        method: 禁止的io机制 取值为 "evport","kqueue","epoll","devpoll","poll","select"
    返回值:
        成功返回 0,失败返回-1
```

----

## 设置libevent支持的features

`int event_config_requeir_features(struct event_config *cfg, int features)`函数功能：

> 设置base_event支持的特性
> EV_FEATURE_ET: 边沿触发模式
> EV_FEATURE_O1: 复杂度O(1)，要求添加、删除单个事件,或者确定哪个事件激活的操作是 O(1)复杂度的I/O机制，select和poll无法实现
> EV_FEATURE_FDS：I/O机制能够处理包括sockets在内的各种文件描述符
> EV_FEATURE_EARLY_CLOSE：检查事件连接是否关闭，可以使用此特性来检测连接是否关闭，而不需要读取此连接状态

```c
enum event_method_feature {
    /** Require an event method that allows edge-triggered events with EV_ET. */
    EV_FEATURE_ET = 0x01,
    /** Require an event method where having one event triggered among
     * many is [approximately] an O(1) operation. This excludes (for
     * example) select and poll, which are approximately O(N) for N
     * equal to the total number of possible events. */
    EV_FEATURE_O1 = 0x02,
    /** Require an event method that allows file descriptors as well as
     * sockets. */
    EV_FEATURE_FDS = 0x04,
    /** Require an event method that allows you to use EV_CLOSED to detect
     * connection close without the necessity of reading all the pending data.
     *
     * Methods that do support EV_CLOSED may not be able to provide support on
     * all kernel versions.
     **/
    EV_FEATURE_EARLY_CLOSE = 0x08
};
```

----

## 设置libevent支持的flag

`int event_config_set_flag(struct event_config *cfg, int flag)`函数功能：

> 设置配置的标志位
> EVENT_BASE_FLAG_NOLOCK：不使用锁，导致使用event_base线程不安全，减少锁的开销
> EVENT_BASE_FLAG_IGNORE_ENV：禁止检查环境？有什么用？
> EVENT_BASE_FLAG_NO_CACHE_TIME：base不缓存事件，缓存时间是为了避免多次调用函数获取时间影响性能
> EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST：如果I/O机制采用的是epoll，可以通过changelist减少系统调用，提高性能，但是可能引入bug？
> EVENT_BASE_FLAG_PRECISE_TIMER：使用高精度的定时器如CLOCK_MONOTONIC

```c
enum event_base_config_flag {
	/** Do not allocate a lock for the event base, even if we have
	    locking set up.

	    Setting this option will make it unsafe and nonfunctional to call
	    functions on the base concurrently from multiple threads.
	*/
	EVENT_BASE_FLAG_NOLOCK = 0x01,
	/** Do not check the EVENT_* environment variables when configuring
	    an event_base  */
	EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
	/** Windows only: enable the IOCP dispatcher at startup

	    If this flag is set then bufferevent_socket_new() and
	    evconn_listener_new() will use IOCP-backed implementations
	    instead of the usual select-based one on Windows.
	 */
	EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
	/** Instead of checking the current time every time the event loop is
	    ready to run timeout callbacks, check after each timeout callback.
	 */
	EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,

	/** If we are using the epoll backend, this flag says that it is
	    safe to use Libevent's internal change-list code to batch up
	    adds and deletes in order to try to do as few syscalls as
	    possible.  Setting this flag can make your code run faster, but
	    it may trigger a Linux bug: it is not safe to use this flag
	    if you have any fds cloned by dup() or its variants.  Doing so
	    will produce strange and hard-to-diagnose bugs.

	    This flag can also be activated by setting the
	    EVENT_EPOLL_USE_CHANGELIST environment variable.

	    This flag has no effect if you wind up using a backend other than
	    epoll.
	 */
	EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,

	/** Ordinarily, Libevent implements its time and timeout code using
	    the fastest monotonic timer that we have.  If this flag is set,
	    however, we use less efficient more precise timer, assuming one is
	    present.
	 */
	EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
```