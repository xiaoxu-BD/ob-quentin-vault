# Docker







分清楚 volume 和 bind mount 的区别



```shell
# 看起来像 Volume，其实是 Bind Mount（因为写了绝对路径）
docker run -v /host/data:/container/data nginx

# 看起来像路径，其实是 Docker 自动创建 Volume（因为没写 / 开头）
docker run -v myvol:/container/data nginx

```





```shell
# 明确声明 Volume
docker run --mount type=volume,source=myvol,target=/container/data nginx

# 明确声明 Bind Mount
docker run --mount type=bind,source=/host/data,target=/container/data nginx

```



开发环境使用 bind mount 

生产环境使用 volume





dockerfile

docker compose 



> 更新: 2026-04-20 13:22:16  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/fnm1f31vacbcigrw>