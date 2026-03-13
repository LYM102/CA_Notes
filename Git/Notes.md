## Git
```bash
#当前文件夹叫做工作区（全局）

#放置到暂存区(全局)
git add
#放置到版本库（每个分支都有）
#所以切换分支的时候要先commit原来的分支
git commit

#本地化用户
git config --global user.name "Zhiheng Cai"
git config --global user.email "caizhiheng0111@icloud.com"

#设置颜色ui
git config --global color.ui auto

#生成私钥、密钥
并将公钥复制到github
ssh-keygen -t rsa -C "your_email@example.com"

#测试本地ssh密钥是否可以和github建立安全连接
ssh -T git@github.com

#初始化git仓库
mkdir git-tutorial
cd git-tutorial
git init

#查看是否同步
git status


#查看日志（哪些人什么时候提交了什么）
git log

#查看特定文件（readme.md）提交前后的差别
git log -p README.md

#查看区别
git diff

#查看分支，带*的是当前分支
git branch

#创建新分支feature-A
git branch feature-A 

#切换到别的分支
git checkout feature-A
#做修改后，要记得commit，来声明这个修改是独属于这个分支的

#合并
#主分支为master
#先checkout到master
git checkout master
git merge --no-ff feature-A
```

> This note is from Cai Zhiheng
