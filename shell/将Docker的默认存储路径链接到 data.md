### 将Docker的默认存储路径链接到 data

---

- 1 停止 Docker 服务
```shell
   sudo systemctl stop docker
```

- 2 移动 Docker 默认存储路径到 /data
```shell
   sudo mv /var/lib/docker /data
```

- 3 创建符号链接将 /var/lib/docker 链接到 /data
```shell
   sudo ln -s /data/docker /var/lib/docker
```

- 4 启动 Docker 服务
```shell
   sudo systemctl start docker
```