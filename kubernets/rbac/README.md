# Using RBAC Authorization

基于角色的控制权限(RBAC)通过访问`rbac.authorization.k8s.io`的接口来驱动权限申明，允许管理员动态配置策略。

在1.8版本中，`RBAC`模式是稳定的，使用`rbac.authorization.k8s.io/v1`接口来使用。

如果需要使用`RBAC`权限控制，要在`apiserver`的启动参数中设置`--authorization-mode=RBAC`。

### API Overview

