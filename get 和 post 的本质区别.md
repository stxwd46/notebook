# get 和 post 的本质区别

<a name="u7dqcl"></a>
## [](#u7dqcl)语义之争
<a name="xmo5zt"></a>
#### [](#xmo5zt)1. 安全性
安全性是指该请求不会引起服务端的数据变更。
<a name="9th2qn"></a>
#### [](#9th2qn)2. 幂等性
幂等性是指同一个请求，执行多次和执行一次，结果是一样的
<a name="frw6se"></a>
#### [](#frw6se)3. 可缓存性
可缓存性是指该缓存是否可以被缓存

所以两种请求方式本质上是 「语义」的对比而不是「语法」的对比

GET的语义是请求获取指定的资源。GET方法是安全、幂等、可缓存的（除非有 Cache-ControlHeader的约束）,GET方法的报文主体没有任何语义。

POST的语义是根据请求负荷（报文主体）对指定的资源做出处理，具体的处理方式视资源类型而不同。POST不安全，不幂等，（大部分实现）不可缓存

<a name="dzvhqf"></a>
## [](#dzvhqf)参考资料

1. [https://www.zhihu.com/question/28586791/answer/145424285](https://www.zhihu.com/question/28586791/answer/145424285)



