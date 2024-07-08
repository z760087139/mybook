# 部署调度

## 部署原理

在 karmada api-server 命名空间上创建 deployment ，配置调度策略 propagation policy 后，schedudler 根据策略进行分配，由 controller 实施部署

## Propagation Policy

{% embed url="https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L13" %}
propagation policy type
{% endembed %}
