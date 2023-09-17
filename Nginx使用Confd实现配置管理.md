---
title: Nginx使用Confd实现配置管理
created: '2023-09-16T06:26:16.561Z'
modified: '2023-09-16T11:37:21.948Z'
---

# Nginx使用Confd实现配置管理

Confd是一个轻量级的配置管理工具。通过查询后端存储etcd，redis，consul等，一方面结合配置模板引擎，保持本地配置最新；一方面具备定期探测机制，配置变更自动下发check和reload。

#### 安装
https://github.com/kelseyhightower/confd

https://github.com/etcd-io/etcd

#### 使用场景
```mermaid
graph TB
    subgraph SVR1
    myserver1(MyServer1)
    file1(file)
    confd1(confd)
    end
    myserver1--读取-->file1
    confd1--更新-->file1
    confd1-.通知.->myserver1

    subgraph SVR2
    myserver2(MyServer2)
    file2(file)
    confd2(confd)
    end
    myserver2--读取-->file2
    confd2--更新-->file2
    confd2-.通知.->myserver2

    subgraph SVR3
    myserver3(MyServer3)
    file3(file)
    confd3(confd)
    end
    myserver3--读取-->file3
    confd3--更新-->file3
    confd3-.通知.->myserver3

    config_server("config-server (etcd)")

    config_server--通知-->confd1
    confd1--读取-->config_server
    config_server--通知-->confd2
    confd2--读取-->config_server
    config_server--通知-->confd3
    confd3--读取-->config_server
```

#### confd工作原理
```mermaid
graph TB
  etcd(读取backend数据)
  template(组装template)
  stage_file(生成stage_file)
  compare_dest_file(对比stage_file和dest_file)
  update_dest_file(更新dest_file)
  next(结束或者下一次循环)
  reload_cmd(执行重启命令reload_cmd)

  etcd-->template-->stage_file-->compare_dest_file
  compare_dest_file--有变更-->update_dest_file-->reload_cmd
  compare_dest_file--无变更-->next

```
- confd配置文件默认在`/etc/confd`中，可以通过参数`-confdir`指定。目录中包含两个子目录，分别是：`conf.d` 存放.toml配置文件和 `templates`存放.tmpl配置模版文件。
- confd会先读取`conf.d`目录中的配置文件(toml格式)，然后根据文件指定的模板路径去渲染模板，再执行`check_cmd`和`reload_cmd`。

#### 服务启动
confd支持以`daemon`或者`onetime`两种模式运行

- onetime模式：只会生成一次配置，之后key无论变化不会再生成
```sh
confd -onetime -backend etcdv3 -node http://127.0.0.1:2379
```
- daemon模式：动态监听后端存储的配置变化，根据配置模板动态生成目标配置文件
```sh
confd -watch -backend etcdv3 -node http://127.0.0.1:2379
```

#### 配置实例
架构图
```mermaid
graph LR
    client(用户客户端)
    subgraph 网关层
    nginx(Nginx)
    confd(Confd)
    end
    etcd(Etcd)
    manager(管理系统)
    server1(Server1)
    server2(Server2)
    server3(Server3)

    client==>nginx
    manager-->etcd-->confd-->nginx
    nginx-->server1
    nginx-->server2
    nginx-->server3
```

1. confd创建配置文件
```sh
cat /etc/confd/conf.d/test.conf.toml

[template]
prefix = "/nginx" # 指定前缀便于区分不同confd项目
src = "test.conf.tmpl" # 配置模板相对路径
dest = "/etc/nginx/conf.d/test.conf" # 目标路径
mode = "0664" # 文件权限
keys = [ # 键数组，与模板文件使用的键对应
  "/nginx_test"
] 
check_cmd = "/usr/sbin/nginx -t -c {{.src}}" # 配置检查命令
reload_cmd = "/usr/sbin/nginx -s reload" # 重启nginx服务
```
2. confd创建模板文件
```sh
cat templates/test.conf.tmpl

upstream www_{{getv "/nginx/www/server/server_name"}} {   
  {{range getvs "/nginx/www/upstream/*"}}       
    server {{.}};  
  {{end}}
}
server { 
  server_name {{getv "/nginx/www/server/server_name"}};  
  location / {       
    proxy_pass http://www_{{getv "/nginx/www/server/server_name"}};     proxy_set_header Host $host;  
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;   
    proxy_set_header X-Forwarded-Proto https;     
    proxy_redirect   off;  
  }
}

```

3. etcd存储具体配置项
```sh
etcdctl set /nginx/https/www/server/server_name test.com
etcdctl set /nginx/https/www/upstream/server1 192.168.1.110
etcdctl set /nginx/https/www/upstream/server2 192.168.1.111
```

4. confd监听etcd配置
```sh
confd -watch -backend etcdv3 -node http://127.0.0.1:2379
```

5. 查看confd生成的nginx配置文件
```sh
 cat /etc/nginx/conf.d/test.conf
 
 upstream www_test.com{        
    server 192.168.1.110;        
    server 192.168.1.111;
 }
 server {   
    server_name test.com;    
    location / {        
      proxy_pass  http://www_test.com;        
      proxy_set_header Host $host;        
      proxy_set_header X-Real-IP $remote_addr;      
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;      
      proxy_set_header X-Forwarded-Proto https;        
      proxy_redirect   off;    
    }
 }

 ```
通过Etcd和Confd来实现nginx upstream的动态更新，简单实现了服务器的灰度发布功能。
