---
title: OTP设计原则-Supervisor Behavior
tags: Erlang OTP Supervisor
article_header:
  type: cover
  image:
    src: /assets/images/20200518-2.jpg
---



## 1、监控原则

- **监控者（supervisor）**负责开启、终止和监控它的子进程。监控者的思想就是通过必要时的**重启**，来保证子进程**一直活着**。
- 子进程规格说明制定了要启动和监控的子进程。
  - 子进程根据规格列表依次启动。
  - 终止顺序和启动顺序**相反**（启动时，后者可能依赖前者，若先终止前者会导致后者直接挂掉）。



## 2、示例

- 启动`gen_server`子进程的监控树：

  ```erlang
  -module(ch_sup).
  -behaviour(supervisor).
  
  -export([start_link/0]).
  -export([init/1]).
  
  start_link() ->
      supervisor:start_link(ch_sup, []).
  
  init(_Args) ->
      SupFlags = #{strategy => one_for_one, intensity => 1, period => 5},
      ChildSpecs = [#{id => ch3,
                      start => {ch3, start_link, []},
                      restart => permanent,
                      shutdown => brutal_kill,
                      type => worker,
                      modules => [ch3]}],
      {ok, {SupFlags, ChildSpecs}}.
  ```



## 3、supervisor flag

- 即示例中的`SupFlags`，定义如下：

  ```erlang
  sup_flags() = #{strategy => strategy(),         % optional
                  intensity => non_neg_integer(), % optional
                  period => pos_integer()}        % optional
      strategy() = one_for_all
                 | one_for_one
                 | rest_for_one
                 | simple_one_for_one
  ```

  - `strategy`：重启策略
  - `intensity`+`period`：最大重启频率



## 4、重启策略

- 重启策略由`init`返回的`map`中的`strategy`指定，默认为`one_for_one`。

- **one_for_one**

  如果子进程终止，只有终止的子进程会重启。

  ![img](https://images2017.cnblogs.com/blog/1046797/201712/1046797-20171225112154444-238345665.png)

- **one_for_all**

  如果子进程终止，其它子进程都会被终止，然后重启所有子进程。

  ![img](https://images2017.cnblogs.com/blog/1046797/201712/1046797-20171225112402131-1112750349.png)

- **rest_for_one**

  如果子进程终止，启动顺序在该子进程后面的子进程都会被终止，随后被重启。

- **simple_one_for_ine**

  简化版的`one_for_one`，所有子进程都是同一个进程动态添加到监控树的实例。



## 5、最大重启频率

- `init`函数返回的`supervisor flag`中的`intensity`和`period`共同限制了子进程给定时间间隔内的重启次数。

  ```erlang
  SupFlags = #{intensity => MaxR, period => MaxT, ...}
  ```

  - `period`：周期时间间隔，默认值为5
  - `intensity`：重启次数，默认值为1

- 当监控者的子进程重启频率超过阈值

  - 监控者终止所有子进程
  - 监控者退出，返回的理由是`shutdown`
  - 该监控者的上一级监控者也会重启或者退出

- 这个重启机制的目的是为了防止进程反复因为**同一个原因**终止和重启。