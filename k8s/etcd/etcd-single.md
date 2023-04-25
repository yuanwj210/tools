### 单台etcd docker部署
```
docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -v /data/etcd:/var/lib/etcd \
 -p 4001:4001 -p 2380:2380 -p 2379:2379 \
 --name etcd quay.io/coreos/etcd:v3.5.4  \
 /usr/local/bin/etcd  \
 -name etcd0 \
 -advertise-client-urls http://172.31.19.4:2379,http://172.31.19.4:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://172.31.19.4:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster etcd0=http://172.31.19.4:2380 \
 -initial-cluster-state new
 ```
### 备份
`docker exec -it  6f0021098b70  etcdctl snapshot save /opt/etcd.db`
### 恢复
`etcdctl snapshot restore  `

### docker 镜像上传
```https://blog.csdn.net/u012856866/article/details/122956380
 
docker tag ubuntu:20.04 image.ankr.com/ankrnetwork/ubuntu:20.04
docker pull image.ankr.com/ankrnetwork/ubuntu:20.04

docker tag SOURCE_IMAGE[:TAG] image.ankr.com/ankrnetwork/REPOSITORY[:TAG]
docker push image.ankr.com/ankrnetwork/REPOSITORY[:TAG]
```
### 从容器复制
`docker cp `
