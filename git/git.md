**git常用命令** <br>
克隆项目
`git clone git@github.com:FBing/design-patterns.git` <br>
查看分支
`git branch` <br>
查看远程分支
`git branch -r `<br>
查看所有分支
`git branch -a` <br>
切换到新的分支
`git checkout [branch name] `<br>
创建+切换分支
`git checkout -b [branch name]` <br>
将新分支推送到github
`git push origin [branch name] `<br>
删除本地分支
`git branch -d [branch name]` <br>
删除github远程分支
`git push origin :[branch name]` <br>
github合并分支
```flow
git checkout (master) 
git pull
git branch -a
git merge (branch)
git status
git add . 
git commit -m "备注内容"
本地仓库代码提交远程仓库
git push 或
git push origin master
```
git merge err refusing to merge unrelated histories解决办法
`git pull origin master --allow-unrelated-histories`

[git command](https://m.geekku.com/spec/github/1422.html )  
 
---

```
docker build -t  镜像名 . #打包构建镜像

docker tag SOURCE_IMAGE[:TAG] image.ankr.com/ankrnetwork/REPOSITORY[:TAG]
docker tag ubuntu:20.04 image.ankr.com/ankrnetwork/ubuntu:20.04

docker pull HOSTNAME/PROJECT-ID/IMAGE:TAG
docker pull image.ankr.com/ankrnetwork/ubuntu:20.04

docker push image.ankr.com/ankrnetwork/REPOSITORY[:TAG]
```





