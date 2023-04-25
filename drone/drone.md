### drone deploy
```
docker run   --volume=/var/lib/drone_ape:/data   --env=DRONE_GITHUB_CLIENT_ID=bdbd0df3e9b363edf489   --env=DRONE_GITHUB_CLIENT_SECRET=778b6206faf05c95037b7974b3
9699097e4ae6d6   --env=DRONE_RPC_SECRET=83bb3d75f54099e293e4952539f86654   --env=DRONE_SERVER_HOST=test.address.com   --env=DRONE_SERVER_PROTO=https   --publish=
8080:80   --restart=always   --detach=true   --name=drone_ape   drone/drone:2

docker run   --volume=/var/lib/drone:/data   --env=DRONE_GITHUB_CLIENT_ID=4ac423d15f6a50c71421   --env=DRONE_GITHUB_CLIENT_SECRET=7e636adcd41d34b36765fe38b0848c
9fe8c7fbb5   --env=DRONE_RPC_SECRET=ae146abd40107879e580fff1dabb9ef5   --env=DRONE_SERVER_HOST=test.company.com   --env=DRONE_SERVER_PROTO=https   --publish=80:80   --pu
blish=443:443   --restart=always   --detach=true   --name=drone   drone/drone:2
```

**安装docker环境**
```
cat >install-docker.sh 
#!/bin/bash
# Install
sudo apt-get update
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update

sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release -y

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io  -y

# install docker-compose v2
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.11.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod 755 /usr/local/bin/docker-compose
```

**drone .drone.yml示例**

example01 后端
```yaml
kind: pipeline
name: default
steps:
  - name: build app DEV
    image: plugins/docker
    settings:
      repo: testnetwork/${DRONE_REPO_NAME}
      tags: [ "${DRONE_COMMIT_SHA:0:7}", "latest" ]
      username:
        from_secret: registry_user
      password:
        from_secret: registry_passwd
    when:
      event:
        - push
      branch:
        - main

  - name: deploy app DEV
    image: appleboy/drone-ssh:1.6.6-linux-amd64
    settings:
      host:
        - 127.0.0.1
      username: root
      key:
        from_secret: ape_mategame_dev
      port: 22
      command_timeout: 2m
      script:
        - docker stop ${DRONE_REPO_NAME}
        - docker rm ${DRONE_REPO_NAME}
        - docker run --pull always -d --name ${DRONE_REPO_NAME} -p 9098:80 ankrnetwork/${DRONE_REPO_NAME}:latest
    when:
      event:
        - push
      branch:
        - main
```
example2 前端
```
kind: pipeline
type: docker
name: cdbc-dev-webfront

steps:
  - name: frontend-dev
    image: node:16
    commands:
      - pwd
      - node -v
      - yarn -v
      - yarn config set registry https://registry.npm.taobao.org/ -g
      - yarn install
      - yarn build-dev
    when:
      event:
        - push
      branch:
        - dev

  - name: sync-cdbc-dev
    image: drillster/drone-rsync
    settings:
      user: root
      key:
        from_secret: do-nginx02-rsa
      hosts:
        - 127.0.0.1
      # 来源项目目录
      source: ./build/*
      # 目标服务器目录
      target: /data/cdbc-dev
    when:
      event:
        - push
      branch:
        - dev
```
