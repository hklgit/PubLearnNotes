### 1. 设置配置选项

第一次运行Airflow时，它将在`$AIRFLOW_HOME`目录中创建一个名为`airflow.cfg`的文件（默认情况下为~/airflow）。此文件包含Airflow的配置，你可以编辑它来更改配置。你还可以使用以下格式为环境变量设置选项：`$AIRFLOW__{SECTION}__{KEY}`（注意双下划线）。

例如，元数据数据库连接字符串可在airflow.cfg如下设置：
```
sql_alchemy_conn = mysql://root:root@localhost:3306/airflow
```
或通过创建相应的环境变量：
```
AIRFLOW__CORE__SQL_ALCHEMY_CONN=mysql://root:root@localhost:3306/airflow
```
你还可以在运行时通过将`_cmd`添加到如下key中来获取连接字符串：
```
sql_alchemy_conn_cmd = bash_command_to_run
```

但是只有三个这样的配置元素，即`sql_alchemy_conn`，`broker_url`和`celery_result_backend`可以作为一个命令获取。背后的思想是不将密码存储在纯文本文件的框上。 优先顺序如下 -
- 环境变量
- 在airflow.cfg中的配置
- 在airflow.cfg中的命令
- 默认值


### 2. 设置后端

如果想要测试Airflow的驱动器，应该考虑设置一个真正的数据库后端并切换到`LocalExecutor`。由于Airflow是使用SqlAlchemy库与其元数据交互构建的，所以可以使用任何支持SqlAlchemy的数据库。我们建议使用`MySQL`或`Postgres`。

**备注**
```
如果你决定使用Postgres，我们建议你使用psycopg2驱动器并在SqlAlchemy连接字符串中指定它。还要注意的是，由于SqlAlchemy没有一种在Postgres连接URI中定位指定的schema的方法(since SqlAlchemy does not expose a way to target a specific schema in the Postgres connection URI)，你需要使用如下命令为你的Role创建一个默认的schema:
ALTER ROLE username SET search_path = airflow, foobar
```
一旦设置了你的数据库，你需要去修改配置文件`$AIRFLOW_HOME/aviation.cfg`中的SqlAlchemy连接字符串(下面配置使用的MySQL数据库):
```
sql_alchemy_conn = mysql://root:root@localhost:3306/airflow
```
还应该将“executor”设置更改为使用“LocalExecutor”，该执行程序可以在本地并行化任务实例:
```
executor = LocalExecutor
```
配置之后，重新实例化数据库:
```
# initialize the database
airflow initdb
```

### 3. Connections

Airflow需要知道如何连接到你的环境中。链接其他系统和服务的主机名，端口，登录名和密码等信息都在UI的`Admin->Connection`中管理。你将使用的管道代码引用Connection对象的conn_id。

![img](http://airflow.incubator.apache.org/_images/connections.png)

默认情况下，Airflow以纯文本形式将连接的密码保存在元数据数据库中。在安装过程中强烈建议使用crypto包。crypto包确实要求你的操作系统安装了libffi-dev。

Airflow管道中的连接可以使用环境变量创建。如果要正确使用连接，环境变量需要使用Airflow的前缀`AIRFLOW_CONN_`，其值为URI格式。

### 4. 使用Celery扩展

`CeleryExecutor`是一种可以扩大worker数目的方法。为此，你需要设置一个Celery后端（RabbitMQ，Redis，...）并更改`airflow.cfg`配置文件以将`executor`指向`CeleryExecutor`并提供相关的Celery配置。

有关设置Celery代理的更多信息，请参阅关于本主题的详尽的[Celery文档](http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html)。

以下是你worker的一些必要条件：
- 需要安装Airflow，并且CLI需要在路径中
- Airflow中的Airflow配置设置应该是相同的
- 在worker执行的operators需要有上下文需要的依赖。例如，如果使用`HiveOperator`，则需要安装Hive CLI，或者如果使用`MySqlOperator`，则需要在PYTHONPATH中提供所需的Python库
- 

### 5. 日志

### 6. 在Mesos上扩展

### 7. 与systemd集成

### 8. 与upstart集成

### 9. 测试模式
