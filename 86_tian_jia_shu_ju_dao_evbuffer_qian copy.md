# 8.6 添加数据到evbuffer前

```c
// 添加数据到evbuffer的前面
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);


// 添加数据到evbuffe
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```