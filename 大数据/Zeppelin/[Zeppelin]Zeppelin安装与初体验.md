### 1.安装
#### 1.1 下载

[点击进入下载页面](http://zeppelin.apache.org/download.html)

==备注==

下载页面会提供两种二进制包：
- zeppelin-0.7.1-bin-netinst.tgz 默认只会提供Spark的Interpreter
- zeppelin-0.7.1-bin-all.tgz 会提供各种各样的Interpreter(MySQL,ElasticSearch等等)

所以说要根据你的使用场景具体选择哪种二进制包．

#### 1.2 解压缩

```
xiaosi@yoona:~$ tar -zxvf zeppelin-0.7.1-bin-netinst.tgz -C opt/
```
### 2. 命令行启动Zepperlin 

#### 2.1 启动Zepperlin

```
bin/zeppelin-daemon.sh start
```
启动成功之后，在浏览器中访问： http://localhost:8080

==备注==

Zepperlin服务器默认端口号为8080

#### 2.2 停止Zepperlin

```
bin/zeppelin-daemon.sh stop
```
### 3. 服务管理器启动Zepperlin

Zeppelin可以使用init脚本自动启动作为一个服务(例如，由`upstart`管理的服务)。

以下是upstart脚本的一个示例，保存为/etc/init/zeppelin.conf，这也允许使用如下命令行方式来管理服务：
```
sudo service zeppelin start  
sudo service zeppelin stop  
sudo service zeppelin restart
```
其他`service managers`可以使用类似的方法，传递`upstart`参数到 `zeppelin-daemon.sh` 脚本中：

```
bin/zeppelin-daemon.sh upstart
```
zeppelin.conf：
```
description "zeppelin"

start on (local-filesystems and net-device-up IFACE!=lo)
stop on shutdown

# Respawn the process on unexpected termination
respawn

# respawn the job up to 7 times within a 5 second period.
# If the job exceeds these values, it will be stopped and marked as failed.
respawn limit 7 5

# 在本例中，zeppelin安装在/home/xiaosi/opt/zeppelin-0.7.1-bin-netinst
chdir /home/xiaosi/opt/zeppelin-0.7.1-bin-netinst
exec bin/zeppelin-daemon.sh upstart
```
### 4. Zeppelin UI

#### 4.1 首页

你第一次连接到Zeppelin，你将会在主页面上看到如下：

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/homepage.png)

页面左侧列出所有现有的笔记。 这些笔记默认存储在`$ZEPPELIN_HOME/Notebook`文件夹中。

可以使用输入文本形式通过名称过滤笔记。还可以创建一个新的笔记，刷新现有笔记的列表（主要考虑手动将它们复制到`$ZEPPELIN_HOME/Notebook`文件夹中的情况）并导入笔记。

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/notes_management.png)

单击`Import Note`连接时，将打开一个新对话框。在对话框中可以从本地磁盘或从远程位置导入你的笔记(如果您提供的URL)。

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/note_import_dialog.png)

默认情况下，导入的笔记的名称与原始笔记相同，但可以通过提供新的名称来覆盖原始名称。

#### 4.2 菜单

##### 4.2.1 Notebook

笔记本菜单提供了与主页中的笔记管理部分几乎相同的功能。 从下拉菜单中，您可以：

- 打开一个特定笔记
- 按名称过滤笔记
- 创建一个新笔记

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/notebook_menu.png)


##### 4.2.2 设置

此菜单可让您访问设置并显示有关Zeppelin的信息。 如果使用默认shiro配置，用户名设置为`anonymous`。 如果要设置身份验证，请参阅[Shiro身份验证](http://zeppelin.apache.org/docs/0.6.0/security/shiroauthentication.html)。

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/settings_menu.png)

#### 4.3 Interpreter

在此菜单中，您可以：
- 配置现有的`interpreter`实例
- 添加/删除`interpreter`实例

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/interpreter_menu.png)

#### 4.4 Configuration

此菜单显示配置文件`$ZEPPELIN_HOME/conf/zeppelin-site.xml`中设置的所有Zeppelin配置:

![image](http://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/ui-img/configuration_menu.png)




原文：
http://zeppelin.apache.org/docs/0.6.0/install/install.html

http://zeppelin.apache.org/docs/0.6.0/quickstart/explorezeppelinui.html
