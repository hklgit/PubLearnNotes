
zookeeper-acl-access-permission-control-mechanism

ZooKeeper 的 ACL 权限控制和 Unix/Linux 操作系统的ACL有一些区别，我们可以从三个方面来理解 ACL 机制，分别是：权限模式(Scheme)、授权对象(ID)和权限(Permission)，通常使用 `scheme:id:perm` 来标识一个有效的ACL信息。

需要注意的是，ACL仅与指定 `ZNode` 有关，不适用于子节点。例如，如果 `/app` 节点仅可由 `ip:172.16.16.1` 读取，而 `/app/status` 设置为 world 模式，那么任何人都可以读取 `/app/status`。不跟我们想象一样，ACL不是递归的。

## 1. 权限

权限(perm)就是指那些通过权限检查后可以被允许执行的操作。在 ZooKeeper 中，所有对数据的操作权限分为以下五大类：

| 权限 | ACL简写 | 描述 |
| --- | --- | --- |
| CREATE | C | 子节点的创建权限，允许授权对象在该数据节点下创建子节点。|
| DELETE | D | 子节点的删除权限，允许授权对象删除该数据节点的子节点。|
| READ | R | 数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或子节点列表等。|
| WRITE | W | 数据节点的更新权限，允许授权对象对该数据节点进行更新操作。|
| ADMIN | A | 数据节点的管理权限，允许授权对象对该数据节点进行ACL相关的设置操作。|

## 2. 授权对象

授权对象(id)指的是权限赋予的用户或一个指定实体，例如IP地址或是机器等。在不同的权限模式下，授权对象是不同的，下表中列出了各个权限模式和授权对象之间的对应关系。

| 权限模式 | 授权对象Id |
| --- | --- |
| IP | 通常是一个IP地址或是IP段，例如 '192.168.0.110”或“192.168.0.1/24' |
| Digest | 自定义，通常是 'username:BASE64(SHA-1(username:password))'，例如 'user2:lo/iTtNMP+gEZlpUNaCqLYO3i5U=' |
| World | 只有一个ID：'anyone' |
| Auth | 该模式不关注授权对象，但必须有 |
| Super | 与Digest模式一致 |


## 3. ACL管理

权限相关命令:
| 命令 | 使用方式 | 描述 |
| --- | --- | --- |
| getAcl | getAcl <path> | 读取 ACL 权限 |
| setAcl | setAcl <path> <acl> | 设置 ACL 权限 |
| addauth | addauth <scheme> <auth> | 添加认证用户 |

### 3.1 设置ACL

通过 zkCli 脚本登录 ZooKeeper 服务器后，可以通过两种方式进行 ACL 的设置。一种是在数据节点创建的同时进行 ACL 权限的设置，命名格式如下：
```
create [-s] [-e] path data acl
```
具体使用如下所示:
```
[zk: 127.0.0.1:2181(CONNECTED) 1] create -e /test/auth-node 'auth scheme node' digest:user1:XDkd2dsEuhc9ImU3q8pa8UOdtpI=:cwrda
Created /test/auth-node
```
另一种方式则是使用 `setAcl` 命名单独对已经存在的数据节点进行 ACL 设置：
```
setAcl path acl
```
具体使用如下所示:
```
[zk: 127.0.0.1:2181(CONNECTED) 2] setAcl /test/auth-node auth:id:crdwa
cZxid = 0x202d
ctime = Sun Sep 22 20:31:31 CST 2019
mZxid = 0x202d
mtime = Sun Sep 22 20:31:31 CST 2019
pZxid = 0x202d
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x100009088560218
dataLength = 16
numChildren = 0
```
### 3.2 获取ACL

适用如下方式获取ACL信息：
```
getAcl <path>
```
具体使用如下所示：
```
[zk: 127.0.0.1:2181(CONNECTED) 3] getAcl /test/auth-node
'digest,'user1:XDkd2dsEuhc9ImU3q8pa8UOdtpI=
: cdrwa
```

## 4. 权限模式

权限模式用来确定权限验证中的校验策略。在 ZooKeeper 中，开发人员使用最多的就是以下五种权限模式。

#### 3.1 IP模式

IP模式通过IP地址粒度来进行权限控制，例如配置了`ip:192.168.0.110`，即表示权限控制都是针对这个IP地址的。同时，IP模式也支持按照网段的方式进行配置，例如`ip:192.168.0.1/24`表示针对`192.168.0.*`这个IP段进行权限控制。

#### 3.2 Digest模式

Digest 是最常用的权限控制模式，也更符合我们对于权限控制的认识，其类似于 `username:password` 形式的权限标识来进行权限配置，便于区分不同应用来进行权限控制。

当我们通过 `username:password` 形式配置了权限标识后，ZooKeeper 会对其先后进行两次编码处理，分别是SHA-1算法加密和BASE64编码，其具体实现由 `DigestAuthenticationProvider.generateDigest(String idPassword)` 函数进行封装，下面代码所示为使用该函数进行`username:password`编码的一个实例:

```java
public class DigestAuthenticationProviderUsage {
    public static void main(String[] args) {
        try {
            System.out.println(DigestAuthenticationProvider.generateDigest("foo:zk-book"));
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
    }
}
```

和Auth模式相比，有两点不同：
- 第一不需要预先添加认证用户(但是在zkCli访问的时候，肯定还是要添加认证用户的)。
- 第二密码是经过sha1及base64处理的密文。
- 授权是针对单个特定用户。
- setAcl使用的密码不是明文，是sha1摘要值，无法反推出用户密码内容。

密码可以通过如下shell的方式生成：
```
echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64
```

### 3.3 World模式

World 是一种最开放的权限控制模式，从其名字中也可以看出，事实上这种权限控制方式几乎没有任何作用，数据节点的访问权限对所有用户开放，即所有用户都可以在不进行任何权限校验的情况下操作 ZooKeeper 上的数据。另外，World 模式也可以看作是一种特殊的 Digest 模式，它只有一个权限标识，即 `world:anyone`。World 模式只有一个授权对象(`anyone`)，表示世界上任意用户。

我们使用如下命令来设置任何人都可以访问的节点：
```
setAcl /newznode world:anyone:crdwa
```
通过正确执行上述操作，我们可以得到如下所示的响应：
![]()

> world模式创建节点的默认模式。

#### 3.4 Auth模式

Auth 是一种特殊模式，不会对任何授权对象ID进行授权，而是对所有已经添加认证的用户进行授权。持久化 ACL 信息时，ZooKeeper 服务器会忽略 `scheme:id:perm` 中提供的任何授权对象表达式。但是仍必须在 ACL 中提供表达式，可以是一个空串''，或者其他任意字符串，因为 ACL 必须与 `scheme:id:permission` 格式匹配:
```
[zk: 127.0.0.1:2181(CONNECTED) 9] setAcl /test/auth-node auth:crdwa
auth:crdwa does not have the form scheme:id:perm
```
如果没有添加身份认证的用户，那么使用 Auth 模式设置 ACL 会报错:
```
[zk: 127.0.0.1:2181(CONNECTED) 3] setAcl /test/auth-node auth::crdwa
Acl is not valid : /test/auth-node
```

### 1.4 Super模式

Super模式，顾名思义就是超级用户的意思，为管理员所使用，这也是一种特殊的 Digest 模式。在 Super 模式下，超级用户可以对任意 ZooKeeper 上的数据节点进行任何操作，不会被任何节点的 ACL 所限制。

### 3.2 Super模式的用法

根据ACL权限控制的原理，一旦对一个数据节点设置了ACL权限控制，那么其他没有被授权的ZooKeeper客户端将无法访问该数据节点，这的确很好的保证了ZooKeeper的数据安全。但同时，ACL权限控制也给ZooKeeper的运维人员带来了一个困扰：如果一个持久数据节点包含了ACL权限控制，而其创建者客户端已经退出或已不再使用，那么这些数据节点该如何清理呢？这个时候，就需要在ACL的Super模式下，使用超级管理员权限来进行处理了。要使用超级管理员权限，首先需要在ZooKeeper服务器上开启Super模式，方法是在ZooKeeper服务器启动的时候，添加如下系统属性：
```
-Dzookeeper.DigestAuthenticationProvider.superDigest=foo:kWN6aNSbjcKWPqjiV7cg0N24raU=
```
其中，“foo”代表了一个超级管理员的用户名；“kWN6aNSbjcKWPqjiV7cg0N24raU=”是可变的，由ZooKeeper的系统管理员来进行自主配置，此例中使用的是“foo:zk-book”的编码。完成对ZooKeeper服务器的Super模式的开启后，就可以在应用程序中使用了，下面是一个使用超级管理员权限操作ZooKeeper数据节点的示例程序。
```java
public class AuthSample_Super {

    final static String PATH = "/zk-book";
    public static void main(String[] args) throws Exception {
        ZooKeeper zooKeeper1 = new ZooKeeper("127.0.0.1:2181", 5000, null);
        zooKeeper1.addAuthInfo("digest", "foo:true".getBytes());
        // 判断是否为空
        if (zooKeeper1.exists(PATH, false) == null) {
            zooKeeper1.create(PATH, "init".getBytes(), Ids.CREATOR_ALL_ACL, CreateMode.EPHEMERAL);
        }
        // 用管理员权限
        ZooKeeper zooKeeper2 = new ZooKeeper("127.0.0.1:2181", 500000, null);
        zooKeeper2.addAuthInfo("digest", "foo:zk-book".getBytes());
        System.out.println(zooKeeper2.getData(PATH, false, null));
        // 用其他用户访问
        ZooKeeper zooKeeper3 = new ZooKeeper("127.0.0.1:2181", 50000, null);
        zooKeeper3.addAuthInfo("digest", "foo:false".getBytes());
        System.out.println(zooKeeper3.getData(PATH, false, null));
    }
}
```
从上面的输出结果中，我们可以看出，由于“foo:zk-book”是一个超级管理员账户，因此能够针对一个受权限控制的数据节点zk-book随意进行操作，但是对于“foo:false”这个普通用户，就无法通过权限校验了。


...
https://ihong5.wordpress.com/2014/07/24/apache-zookeeper-setting-acl-in-zookeeper-client/
