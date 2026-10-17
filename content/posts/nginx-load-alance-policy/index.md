+++
date = 2026-10-13
title = "仿照Nginx设计自己的负载均衡策略"
description = ""
slug = ""
authors = []
tags = ["Nginx", "负载均衡", "反向代理"]
series = ["Linux网络编程"]
featuredImage = "assets/cover.png"
toc = true
draft = true
+++

## 1 宏观理解
本文章用到 [nginx 0.1.0](https://nginx.org/download/nginx-0.1.0.tar.gz) 源码。

0.1.0 版本采简单取轮询的方式遍历上游服务器（后端服务器），选择可用服务器。

具体的负载均衡代码位于 `src/event/ngx_event_connect.c` 文件的函数。
```c
int ngx_event_connect_peer(ngx_peer_connection_t *pc)
```

返回值是表示成功/失败/没有可用连接等情况。

参数是一个结构体，该结构体包含了连接池及其相关的参数。

这个函数的主要逻辑是 轮询 + 错误次数 + 惩罚超时的逻辑。

在宏观角度可以看作，从一个下标开始死循环遍历上游服务器，直到选出一个可用的上游服务器。

`Nginx`，认为所有的上游服务器都是可用的，当选择了一个上游服务器后，发现不能工作，错误次数就加1，并记录错误时间。

如果这个错误次数超过了容忍度，并且上一次的错误时间没有超过最大惩罚超时。也就是说错误次数达到一定次数，并且短时间内发生过错误，那么就认为该服务器暂时不可用，轮询遍历下一个机器。

当然，如果错误次数没有超过容忍度，认为该机器可用。如果上次错误时间超过了惩罚超时，则该机器之前发生过错误，但是现在可能能使用了。

## 2 源码解读
由于源码太长了，这里简化成主要逻辑。

### 2.1 只有一台机器
```c
ngx_peer_t          *peer;
now = ngx_time(); 
```
`peer` 指向最终选择的后端服务器。
`now` 获取当前时间，用于判断超时失败。

```c
if (pc->peers->last_cached) {
  
    // 连接池有空闲的连接
  
    return NGX_OK;  // 复用成功
}
```
尝试使用连接池中的空闲连接，提高效率。

```c
if (pc->peers->number == 1) {
    peer = &pc->peers->peers[0];
}
else
{
    // 有多台机器
}
```
只有一台服务器，就用这个服务器。

### 2.2 多台服务器选择策略
下面的多台机器就是轮询的核心算法逻辑：
```c
if (pc->tries == pc->peers->number) {
    pc->cur_peer = pc->peers->current++;

    if (pc->peers->current >= pc->peers->number) {
        pc->peers->current = 0;  // 就像数组循环队列一样
    }
}
```
`tries` 是剩余尝试次数，如果等于上游服务器数量，则说明是第一次尝试。这里也表明了，`nginx` 不会随意丢弃请求，而是多次尝试。

这里的 current 是全局轮询指针，下次轮询从current开始。cur_peer是当前这次轮询指向的上游服务器指针。

```c
for ( ;; ) {
  peer = &pc->peers->peers[pc->cur_peer];

  // 失败次数没有超过容忍度或者过了冷却时间，认为机器可用
  if (peer->fails <= pc->peers->max_fails
      || (now - peer->accessed > pc->peers->fail_timeout))
  {
      break;
  }

  // 选择下一台
  pc->cur_peer++;

  if (pc->cur_peer >= pc->peers->number) {
      pc->cur_peer = 0;
  }

  // 尝试次数减一
  pc->tries--;

  if (pc->tries == 0) {
      // 返回失败
      return NGX_ERROR;
  }
}
```

当然后面还会有真正创建socket，绑定连接等行为，这里不做探究。

## 3 自己设计负载均衡
```cpp
class Machine
{
public:
    Machine(const std::string& ip, uint16_t port);
    void IncLoad();           // 增加负载
    void DecLoad();           // 减少负载
    void IncFails();          // 增加失败次数
    void ResetFails();        // 重置失败次数
    void UpdateLastFailTime();// 更新上次失败时间为当前时间
    void ResetLastFailTime(); // 重置上次失败时间
    uint64_t Load();          // 获取负载
    int Fails();              // 获取失败次数
    int LastFailTime();       // 获取上次失败时间
public:
    std::string _ip;            // ip
    uint16_t _port;             // 端口
    uint64_t _load;             // 负载
    std::unique_ptr<std::mutex> _mtx_load;          // 保护load
    std::unique_ptr<std::mutex> _mtx_fails;         // 保护fails
    std::unique_ptr<std::mutex> _mtx_last_fail_time;// 保护_last_fail_time;
    int _fails;                 // 失败次数
    time_t _last_fail_time;     // 上次失败时间
};
```
`Machine` 是对上游机器的描述，其中一堆方法都是对失败次数/上次失败时间/负载值的设置。需要注意加锁。

```cpp
const int g_max_fails = 3;    
const int g_trieds = 5;       
const int g_fail_timeout = 30;

class LoadBalance
{
public:
    LoadBalance()
    {
        // 从配置文件中读取ip 和 port 并且构建 `Machine` 对象，放进 _machines 数组中。
    }

    int SelectMachine();
    
    Machine& operator[](size_t pos)
    {
        assert(pos >= 0);
        return _machines[pos];
    }

private:
    std::vector<Machine> _machines;          // 所有机器
    std::mutex _mtx;                         // 用于选择机器
};
```

主要逻辑在 `SelectMachine` 里面，这个调用这个函数，返回一个 `_machines` 的下标。

```cpp
int SelectMachine()
{
    // 加锁
    std::lock_guard<std::mutex> lg(_mtx);

    int pos = -1;
    uint32_t minLoad = INT_MAX;
    for(int i = 0; i < _machines.size(); ++i)
    {
        // 获取当前时间
        time_t now = time(nullptr);
        // 判断失败次数和惩罚时间
        if(_machines[i].Fails() >= g_max_fails && now - _machines[i].LastFailTime() < g_fail_timeout)
        {
            // 该机器不可用
            continue;
        }
        else if(_machines[i].Fails() >= 3 && now - _machines[i].LastFailTime() >= g_fail_timeout)
        {
            // 认为该机器可能已经能用了
            // 重置失败次数
            _machines[i].ResetFails();
        }
        if(minLoad > _machines[i].Load())
        {
            // 选择最低负载机器
            pos = i;
            minLoad = _machines[i].Load();
        }
    }
    if(pos != -1)
        _machines[pos].IncLoad();

    return pos;
}
```
这里的逻辑和 `Nginx` 的策略有所不同，这个代码是从 0 下标开始的遍历，不是维护一个全局下标。而且，这里是选择负载最低的可用机器，这和具体业务相关，灵活调整。

