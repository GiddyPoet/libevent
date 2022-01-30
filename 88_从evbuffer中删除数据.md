# 8.6 从evbuffer中删除数据

```c
//  从evbuffer中丢弃size个数据，返回值0 表示成功 -1 表示失败
int evbuffer_drain(sturct evbuffer *buf , size_t len);

//  从evbuffer中移除datlen个字节的数据存入data中 
int evbuffer_remove(struct evbuffer *buf,void *data, size_t datlen);

```