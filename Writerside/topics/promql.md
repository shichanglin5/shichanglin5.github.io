# Promql 问题

### 如何检测指标缺失
**描述**：当某个节点停止采集时（target_up指标查询为空），如何发现该节点停止采集  
**思路**  
**1、固定节点场景**  
即指标的ident是已知的，那么可以直接使用absent函数，因为absent函数当指标缺失时会根据函数参数自动生成时序，比如 `absent(target_up)` 那么就会生成 `target_up` 指标，如果是 `absent(target_up{job='xxx'})` 那么就会生成 `target_up{job='xxx'}` 时序，所以就会非常简单，节点缺失的条件是之前（比如1m前）存在而现在（最近1m)不存在，写promeql:  
`present_over_time(target_up{ident="10.200.140.215:9800"}[1m] offset 2m) and on(ident) absent_over_time(target_up{ident="10.200.140.215:9800"}[1m])`  

**2、非固定节点场景**
这种情况下 absent 函数就不适用了，换一种思路，之前存在和现在不存在除了使用 present 和 absent 外，还可以将时序不同时间点（使用offset）进行关联，如果能关联上，那么两个时间点都存在，否则，其中一个时间点不存在，比如   
```Text
`((target_up{ident='abc'}offset 3m - target_up{ident='abc'}) or (target_up{ident='abc'}offset 3m-2))<-1`
```
如果节点 3m 前存在，而现在不存在，那么 `(target_up{ident='abc'}offset 3m - target_up{ident='abc'})` 会为空，这时候就会返回 `or` 后面子句的结果 `(target_up{ident='abc'}offset 3m-3)`，第一个子句`target_up`相减最小结果为`0-1=-1`，而第二个子句返回结果最大为`1-3=-2`，所以最后判断结果如果小于-1，则说明节点丢失  
但这会可能会触发误告警，因为这里计算的结果是某个时间点，不能表示节点丢失的持续性（当容器重启可能会触发误告警）  

如果能够检测持续性呢？上面是对`offset 3m`前和现在进行比较，如果再加个条件，比如 `offset 3m`前和`offset 1m`也能满足上面的关系，那就能说明 `offset 1m` 到现在节点都是出于丢失状态的，这两种状态同时满足，就可以避免重启触发误告警，promql如下:  
```Text
((target_up{job="ns-proxy-stage"} offset 3m - target_up{job="ns-proxy-stage"}) or (target_up{job="ns-proxy-stage"} offset 3m -3))< -1
and on(job,ident)
((target_up{job="ns-proxy-stage"} offset 3m - target_up{job="ns-proxy-stage"} offset 1m) or (target_up{job="ns-proxy-stage"} offset 3m -3))< -1
```
但还有个问题：如果是新增节点，第一个子句 `(target_up{ident='abc'}offset 3m - target_up{ident='abc'})` 也可能返回为空，导致整个 promql 产生误告警  

如果避免？增加个 `present_over_time offset 3m`，完整Promql 如下：
```Text
present_over_time(target_up{job="ns-proxy-stage"}[1m] offset 3m)
and on(job,ident)
((target_up{job="ns-proxy-stage"} offset 3m - target_up{job="ns-proxy-stage"}) or (target_up{job="ns-proxy-stage"} offset 3m -3))< -1
and on(job,ident)
((target_up{job="ns-proxy-stage"} offset 3m - target_up{job="ns-proxy-stage"} offset 1m) or (target_up{job="ns-proxy-stage"} offset 3m -3))< -1
```

### promql 检测 qps 波动
```Text
(delta((sum(requests_total{host=~".*infprometheus.dss.17usoft.com",request_uri=~"/write/api/v1/push", status="204"}offset 1m)by(host, status))[5m]) / quantile_over_time(0.5, sum(requests_total{host=~".*infprometheus.dss.17usoft.com",request_uri=~"/write/api/v1/push", status="204"}offset 1m)by(host, status)[1h]) < -0.15
or
delta((sum(requests_total{host=~".*infprometheus.dss.17usoft.com",request_uri=~"/write/api/v1/push", status="204"}offset 1m)by(host, status))[5m]) / quantile_over_time(0.5, sum(requests_total{host=~".*infprometheus.dss.17usoft.com",request_uri=~"/write/api/v1/push", status="204"}offset 1m)by(host, status)[1h]) > 0.15) * 100

```