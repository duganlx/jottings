# redis 使用

```bash
sudo docker run -p 6379:6379 --restart=always --name redis_queue -itd redis redis-server --requirepass pwd
sudo docker exec -it jhluc_redis redis-cli
redis-cli -h host -p port -a pwd
```