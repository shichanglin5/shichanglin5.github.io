# 用法tips

## 基于vmauth实现多集群故障自动转移
vmauth 可以基于规则配置请求转发，每个规则可以配置多个负载url，多个负载url的负载方式支持`first_available`和`least_loaded`  
**first_available**: 优先转发到顺序靠前的负载
**least_loaded**: 负载均衡

线上环境为了保障高可用性，我们部署多个时序集群，同一份数据至少在两个集群来容灾  

vmauth 作为一个网关组件，在多中心/机房部署，域名配置就近访问，请求到达vmauth后基于规则请求到正确的集群

为了保障查询高可用，对于每个租户的查询负载配置副本集群，如下：  
```yaml
users:
- name: 'read / meta'
  auth_token: meta
  url_map:
  - src_paths:
      - "^/api/v1/.*"
    drop_src_path_prefix_parts: 0
    url_prefix:
      - http://idc1.monitor.com/prometheus
      - http://idc2.monitor.com/prometheus
```
为了减少跨idc网络链路的转发，vmauth在转发请求时优先转发到同中心集群（就近负载）  
对于`idc1`的负载，vmauth永远优先转发到 `http://idc1.monitor.com/prometheus`  
对于`idc2`的负载，vmauth永远优先转发到 `http://idc2.monitor.com/prometheus`

为了达到这种效果，不同idc的vmauth配置负载时优先将就近的负载地址配置在前面，然后备份集群负载配置在后面，然后 vmauth 配置负载策略为 `first_available` 即可
这样当就近集群故障时，自动重试请求到备份集群，保障查询高可用