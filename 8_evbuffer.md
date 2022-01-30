# 8 数据封装evBuffer

libevent 的 evbuffer 实现了为向后面添加数据和从前面移除数据而优化的字节队列。

evbuffer 用于处理缓冲网络 IO 的“缓冲”部分。它不提供调度 IO 或者当 IO 就绪时触发 IO 的 功能:这是 bufferevent 的工作。

除非特别说明,本章描述的函数都在 event2/buffer.h中声明。


## 创建evbuffer

创建和销毁evbuffer。

```c
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```

## 线程安全的evbuffer

创建锁和加锁和解锁操作。
```c
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```


当`evbuffer_enable_locking()`中参数`lock`为空的时候，函数内部会调用`EVTHREAD_ALLOC_LOCK`创建lock。

```c
int
evbuffer_enable_locking(struct evbuffer *buf, void *lock)
{
#ifdef EVENT__DISABLE_THREAD_SUPPORT
	return -1;
#else
	if (buf->lock)
		return -1;

	if (!lock) {
		EVTHREAD_ALLOC_LOCK(lock, EVTHREAD_LOCKTYPE_RECURSIVE);
		if (!lock)
			return -1;
		buf->lock = lock;
		buf->own_lock = 1;
	} else {
		buf->lock = lock;
		buf->own_lock = 0;
	}

	return 0;
#endif
}
```

## 获取evbuffer参数(长度)

获取数据长度`evbuffer_get_length(const struct evbuffer *buf);`，`evbuffer_add(struct evbuffer * buf ,const void *data, size_t datlen);`添加数据到evbuffer里

```c
size_t evbuffer_get_length(const struct evbuffer *buf);

int evbuffer_add(struct evbuffer * buf, const void *data, size_t datlen);
```

## 向evbuffer添加数据

### 添加数据到末尾

```c
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```

### 添加格式化的数据到buf末尾
```c
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...)

int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```