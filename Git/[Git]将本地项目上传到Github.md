### 1 .安装Git

第一步就不用说了，首先需要在本地安装Git

### 2. 创建项目

在Gitlab上创建项目, 点击右上角加号到项目创建页面，填写项目名称，选择项目访问权限，private为授权的用户才能访问．

项目地址为：
```
git@github.com:sjf0115/PubLearnNotes.git
```

### 3. 上传

初始化Git项目
```
xiaosi@yoona:~/code/PubLearnNotes$ git init
```
本地提交
```
xiaosi@yoona:~/code/PubLearnNotes$ git commit -am "ADD:test"
```
关联远程仓库
```
xiaosi@yoona:~/code/PubLearnNotes$ git remote add origin git@github.com:sjf0115/PubLearnNotes.git
```
推送到远程仓库
```
xiaosi@yoona:~/code/PubLearnNotes$ git push origin master
```

git remote add origin git@github.com:sjf0115/HexoBlog.git
