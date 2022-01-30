# 8.6 从evbuffer中拷贝数据

```cpp
// 从buf中拷贝datlen长度的数据到data中，原始数据还保留，和evbuffer_pullup有些许不同，会做数据拷贝？
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);

// 类似evbuffer_copyout的功能只是从evbuffer的中的某个指针地址开始拷贝
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf, const struct evbuffer_ptr *pos, void *data_out, size_t datlen);
```