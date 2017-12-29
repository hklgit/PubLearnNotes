
默认情况下，所有的功能都是开放的。限制访问`Web`应用程序的简单方法是在网络级别或通过使用`SSH`隧道进行。

It is however possible to switch on authentication by either using one of the supplied backends or create your own.

### 1. 密码

最简单的身份验证机制之一是要求用户在登录之前指定密码。密码身份验证需要在需求文件中使用密码子包。 密码散列在存储密码之前使用bcrypt。

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

Airflow提供一个类似插件的方式让使用者可以更好地按自己的需求定制， 这里 列出了官方的一定Extra Packages.

默认安装后是，后台管理的webserver是无须登录，如果想加一个登录过程很简单：

首先安装密码验证模块：
```
sudo pip install airflow[password]
```
有可能在安装的过程中会有一些依赖包没有，只需要相应地装上就可以了。比如 libffi-dev 、 flask-bcrypt

当启用密码验证时，在用户可以登录之前需要创建初始用户凭证。在此认证后端的迁移中未创建初始用户，以防止默认的`Airflow`安装受到攻击。创建一个新用户必须通过在`AirFlow`安装的同一台机器上的`Python REPL`来完成。进入你`airflow`安装目录后运行以下代码，将用户帐号密码信息写入DB：
```
import airflow
from airflow import models, settings
from airflow.contrib.auth.backends.password_auth import PasswordUser
user = PasswordUser(models.User())
user.username = 'xiaosi3'
user.email = '1203745031@qq.com'
user.password = 'zxcvbnm'
session = settings.Session()
session.add(user)
session.commit()
session.close()
exit()
```
重启webserver即可看见登录页面。



原文:http://pythonhosted.org/airflow/security.html#web-authentication
