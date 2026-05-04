# Nginx

| 符号 | 名称 | 含义 | 优先级（由高到低） |
| :--- | :--- | :--- | :--- |
| `=` | **精确匹配** | 必须一模一样，多一个字都不行 | **最高 (1)** |
| `^~` | **前缀匹配（优先）** | 以这个字符串开头，一旦匹配成功，不再检查正则 | **次高 (2)** |
| `~` 或 `~*` | **正则匹配** | `~`区分大小写，`~*`不区分。匹配正则表达式 | **低 (3)** |
| 无符号 | **普通匹配** | 以这个字符串开头 | **最低 (4)** |

> server是 Nginx 监听某个端口（比如 80）的“虚拟服务器”，用来接收请求；location是用来匹配请求的路径（比如 /api/user）；
>
> 而 proxy\_pass就是 Nginx 接收到匹配的请求后，把请求转发（代理）到另一个后端服务地址，这就是所谓的“反向代理”。

```nginx
server {
  listen 80;
  server_name localhost;  //Nginx服务器的地址 (大概是这个意思)

  location /api/user {
    proxy_pass http://192.168.200.100:8080/api/user;  // 符合/api/user的路径会被转发到这个地址
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

nginx基本语法: nginx -t 校验是否语法是否正确

nginx reload

### root 和 alias的区别:

```java
location /images/ {
    root /data/www;
}
```

用户访问http://ip/images/logo.png

其实会去找: /data/www/images/logo.png 这个地方的文件;

如果使用

```java
location /images/ {
    alias /data/pics;
}
```

用户访问: http://ip/images/logo.png

其实会在去找 /data/pics/logo.png

### Nginx 配置优先级

| 符号 | 名称 | 含义 | 优先级（由高到低） |
| :--- | :--- | :--- | :--- |
| `=` | **精确匹配** | 必须一模一样，多一个字都不行 | **最高 (1)** |
| `^~` | **前缀匹配（优先）** | 以这个字符串开头，一旦匹配成功，不再检查正则 | **次高 (2)** |
| `~` 或 `~*` | **正则匹配** | `~`区分大小写，`~*`不区分。匹配正则表达式 | **低 (3)** |
| 无符号 | **普通匹配** | 以这个字符串开头 | **最低 (4)** |


> 更新: 2026-03-01 20:54:09  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/tp4bt3ht6m9hqop1>