# 8.6 从evbuffer中peek数据

## interface
```c
// 从evbuffer中peek siez个数据，用于查看，将其拷贝出来，原始数据还在
unsigned char *evbuffer_pullup(struct evbuffer *buf,ev_ssize_t size);

// 从evbuffer中丢弃size个数据，返回值0 表示成功 -1 表示失败
int evbuffer_drain(struct evbuffer *buf, size_t len);
```

## example

```c
int parse_socks4(struct evbuffer *buf,ev_uint16_t,ev_uint32_t *addr){
    unsigned char *mem;
    mem = evbuffer_pullup(buf,8);

    if(mem == NULL) {
        return 0;
    }else if (mem[0] != 4 || mem[1] !=1) {
        return -1;
    }else {
        memcpy(port,mem+2,2);
        memcpy(addr,mem+4,4);
        *port = ntohs(*port);
        *addr = ntohl(*addr);

        evbuffer_drain(buf,8);
        return 1;
    } 
}
```