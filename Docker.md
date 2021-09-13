### docker镜像拉取加速

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://zhsyfcrd.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### NameSpaces

```bash
sudo ip netns list # 查看命名空间列表
sudo ip netns delete test1 # 删除
sudo ip netns add test1 # 添加
sudo ip netns exec test1 ip a
sudo ip netns exex test1 ip link set dev lo up

sudo ip link add veth-test1 type veth peer name veth-test2
sudo ip link set veth-test1 netns test1
sudo ip link set veth-test1 netns test2
sudo ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1
sudo ip netns exec test1 ip addr add 192.168.1.2/24 dev veth-test1
sudo ip netns exec test1 ip link set dev veth-test1 up
sudo ip netns exec test2 ip link set dev veth-test2 up
sudo ip netns exec test1 ping 192.168.1.2
```



![image-20210115095044831](https://raw.githubusercontent.com/gpasserby/images/master/img/image-20210115095044831.png)

### bridge0

```bash
sudo docker network ls
sudo docker network inspect CONTAINER_ID
```

![image-20210115100833471](https://raw.githubusercontent.com/gpasserby/images/master/img/image-20210115100833471.png)

![image-20210115101023455](https://raw.githubusercontent.com/gpasserby/images/master/img/image-20210115101023455.png)

### link

```bash
sudo docker run -d --name test2 --link test1 ....  # link 是有方向的
```

### 端口映射

```bash
sudo docker run --name web -d -p 80:80 nginx
```

![image-20210119094421205](https://raw.githubusercontent.com/gpasserby/images/master/img/image-20210119094421205.png)

