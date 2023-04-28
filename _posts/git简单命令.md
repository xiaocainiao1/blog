#                                  git简单命令

**上传**

```
git init    初始化仓库

git add .   提交当前目录到缓存

git commit  -m "my first commit"  把缓存提交到本地仓库

git  remote add origin    添加远程仓库

git push -u origin master 推送到远程仓库   --注意按提示输入用户名、密码
```

备注：如果第一次使用Git,**需配置全局的参数**

```
git config --global user.name “你的用户名”
git config --global user.email “你的邮箱”
```

**下载**

 git clone http://www.kernel.org/pub/scm/git/git.git

