## 怎么配置docker

### 1. 常用配置
```json
{
  "insecure-registries": [
    "http://harbor.abc.com"
  ]
}
```
配置后， 重新加载并重启服务。 
```shell
sodo systemctl daemon-reload
sodo systemctl restart docker 
```

### 2. 配置详细说明
