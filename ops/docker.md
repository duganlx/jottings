# docker 操作

- 查看文件挂载 `docker inspect --format '{{ json .Mounts }}' xxx`
- 查看端口映射 `docker port xxx`
- 查看 2024-02-02 后的最新 10 条日志 `docker logs --since="2024-02-02" --tail=10 xxx`; 实时加 `-f`; 30 分钟内 `--since 30m`;
- 拷贝文件 `docker cp /root/a xxx:/root/b`
- 进入容器 `docker exec -it xxx /bin/bash`
- 编译 Dockerfile `docker build -t xxx:v1.1 .`