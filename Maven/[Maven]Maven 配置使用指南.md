---
layout: post
author: sjf0115
title: Maven settings.xml使用指南
date: 2018-02-01 11:29:01
tags:
  - Maven

categories: Maven
permalink: maven-settings
---

### 1. 概述

`settings.xml` 文件中的 `settings` 元素包含用于定义各种值的元素，可以使用不同方式（如 `pom.xml`）配置 `Maven` 的运行，但是不应该捆绑到任何指定的项目上或分发给一个用户。其中值包括本地存储库位置，备用远程存储库服务器以及身份验证信息等。

`settings.xml` 文件可能存在两个位置中：
- Maven安装目录中： `${maven.home}/conf/settings.xml`
- 用户安装目录中： `${user.home}/.m2/settings.xml`

前一个 `settings.xml` 也被称为全局设置，后一个 `settings.xml` 被称为用户设置。如果两个文件都存在，它们的内容将被合并，用户指定的 `settings.xml` 占主导地位。

备注:
```
如果你需要从头开始创建特定用户设置，将全局设置从 Maven 安装目录中复制到 ${user.home}/.m2 目录中是最简单的方法。Maven 的默认 settings.xml文件 是一个包含注释和示例的模板，因此可以快速的调整来满足你的需求。
```

以下是`settings`元素下的顶级元素的概述：
```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline/>
  <pluginGroups/>
  <servers/>
  <mirrors/>
  <proxies/>
  <profiles/>
  <activeProfiles/>
</settings>
```
可以将如下表达式来插入 `settings.xml`文件中：
- `${user.home}`和其他系统配置（从`Maven 3.0`版本开始）
- `${env.HOME}`等环境变量

Note that properties defined in profiles within the settings.xml cannot be used for interpolation.

### 2. 详情

#### 2.1 简单值

一半的顶级 `settings` 元素都是简单值，表示一系列值，描述了全天活跃的构建系统的元素。
```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>${user.home}/.m2/repository</localRepository>
  <interactiveMode>true</interactiveMode>
  <usePluginRegistry>false</usePluginRegistry>
  <offline>false</offline>
  ...
</settings>
```

(1) localRepository

该值表示构建系统本地存储库的路径。默认值是 `${user.home}/.m2/repository`。这个元素对于主构建服务器特别有用，允许所有登录用户从一个本地存储库进行构建。

(2) interactiveMode

`Maven` 是否需要和用户交互以获得输入。如果需要，该值为 `true`，否则为 `false`。默认为 `true`。

(3) usePluginRegistry

`Maven` 是否使用 `${user.home}/.m2/plugin-registry.xml`文件来管理插件版本，如果需要，该值为 `true`，默认为 `false`。请注意，对于`Maven 2.0`的当前版本，不需要 `plugin-registry.xml` 文件。

(4) offline

`Maven` 是否需要在离线模式下运行，如果需要，该值为 `true`，否则为 `false`。由于网络设置或安全原因，此元素对于构建服务器不能连接到远程存储库时非常有用。

#### 2.2 Plugin Groups 插件组

该元素包含一个 `pluginGroup` 元素的列表，每个 `pluginGroup` 都包含一个 `groupId`。当使用一个插件并且在命令行中没有提供 `groupId` 时，会搜索该列表。这个列表自动包含 `org.apache.maven.plugins` 和 `org.codehaus.mojo`。

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <pluginGroups>
    <pluginGroup>org.mortbay.jetty</pluginGroup>
  </pluginGroups>
  ...
</settings>
```

例如，给定上述设置 `Maven` 命令行可以使用如下简短命令来运行 `org.mortbay.jetty：jetty-maven-plugin：run`：
```
mvn jetty:run
```

#### 2.3 Servers 服务器

下载和部署仓库是由 `POM` 的 `repositories` 和 `distributionManagement` 元素定义。但是，某些设置（如用户名和密码）不应该在 `pom.xml` 上配置 。这种类型的信息应该在 `settings.xml` 中构建的 `server` 上存在。

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <servers>
    <server>
      <id>server001</id>
      <username>my_login</username>
      <password>my_password</password>
      <privateKey>${user.home}/.ssh/id_dsa</privateKey>
      <passphrase>some_passphrase</passphrase>
      <filePermissions>664</filePermissions>
      <directoryPermissions>775</directoryPermissions>
      <configuration></configuration>
    </server>
  </servers>
  ...
</settings>
```
(1) id

这是服务器 `server` 的Id（不是登录用户的Id），与 `Maven` 尝试连接的仓库/镜像的 `id` 元素匹配。

(2) username, password

这些元素成对出现，表示对此服务器进行身份验证所需的登录名和密码。

(3) privateKey, passphrase

与前两个元素一样，也是成对出现，指定了私钥的路径（默认为 `${user.home}/.ssh/id_dsa`）以及在需要时提供一个`passphrase`。`passphrase` 和 `password` 元素在不久的将来可能存储在外部，但是目前它们必须在 `settings.xml` 文件中以纯文本的形式出现。

(4) filePermissions, directoryPermissions

如果在部署时创建仓库文件或目录，这些是需要使用权限的（译者注：这两个元素指定了创建的文件或目录的权限）。它们的合法值是一个对应于 `*nix`（`unix`或`linux`） 文件的权限的三位数的数字，例如上述配置中的 `664` 或 `775`。

#### 2.4 Mirrors 镜像

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <mirrors>
    <mirror>
      <id>planetmirror.com</id>
      <name>PlanetMirror Australia</name>
      <url>http://downloads.planetmirror.com/pub/maven2</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>
```

(1) id, name

该镜像的唯一标识符以及名称。`id` 用来区分不同 `mirror` 元素，~~并在连接镜像时从 `<servers>` 章节选择相应的证书~~。

(2) url

该镜像的URL。构建系统使用此URL来连接到仓库，而不是使用原始仓库的URL。

(3) mirrorOf

被镜像仓库的 `id`。例如，指向 `Maven central` 存储仓库（https://repo.maven.apache.org/maven2/）的镜像，就将此元素设置为 `central`。更高级的映射，如 `repo1`，`repo2` 或 `*`，`！inhouse`也是可以的。这不能与镜像 `id` 一致。

有关更深入的镜像介绍，请阅读[镜像设置指南](http://maven.apache.org/guides/mini/guide-mirror-settings.html)。

#### 2.5 Proxies 代理

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <proxies>
    <proxy>
      <id>myproxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>proxy.somewhere.com</host>
      <port>8080</port>
      <username>proxyuser</username>
      <password>somepassword</password>
      <nonProxyHosts>*.google.com|ibiblio.org</nonProxyHosts>
    </proxy>
  </proxies>
  ...
</settings>
```

(1) id

该代理的唯一标识符。这用于区分不同 `proxy` 元素。

(2) active

是否需要激活该代理，如果需要，则为 `true`。这在声明一组代理时是非常有用的，一次只能激活一个代理（译者注：一组代理智能激活一个，可以使用该项来控制激活哪个代理）。

(3) protocol, host, port

代理的 `protocol://host:port`，分隔成离散的元素（`protocol`协议，`host`主机名，`port`端口）以方便配置。

(4) username, password

这些元素成对出现，表示对此代理服务器进行身份验证所需的登录名和密码。

(5) nonProxyHosts

不被代理的主机名列表。列表的分隔符是代理服务器的指定类型（译者注：列表分隔符由代理服务器指定）; 在上面的例子中分隔符为`|`，以逗号分隔也是经常见的。

#### 2.5 Profiles

`settings.xml` 中的 `profile` 元素是 `pom.xml profile` 元素的简化版本。由 `activation`, `repositories`, `pluginRepositories` 和 `properties` 元素组成。`profile` 元素只包含这四个元素，~~因为这里只关心构建系统这个整体（作为 `settings.xml` 文件的角色），而不关心个别项目对象模型的设置~~。

如果 `settings` 中的一个 `profile` 被激活，那么其值将会覆盖 `POM` 或 `profiles.xml` 文件中相同的 `ID` 的 `profile`。

##### 2.5.1  Activation

`Activation` 是 `profile` 的关键。与 `POM` 的 `profile` 一样， `profile` 的作用在于它只能在特定情况下修改一些值。这些特定情况是通过 `activation` 元素指定的（译者注：`activation` 指定了激活 `profile` 的条件）。

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <profiles>
    <profile>
      <id>test</id>
      <activation>
        <!-- profile 默认是否激活的标识 -->
        <activeByDefault>false</activeByDefault>
        <!-- 当匹配的jdk被检测到，profile 被激活 -->
        <jdk>1.5</jdk>
        <os>
          <!-- 激活 profile 的操作系统的名字 -->
          <name>Windows XP</name>
          <!-- 激活 profile 的操作系统所属家族 -->
          <family>Windows</family>
          <!-- 激活 profile 的操作系统体系结构 -->
          <arch>x86</arch>
          <!-- 激活 profile 的操作系统版本 -->
          <version>5.1.2600</version>
        </os>
        <property>
          <!--激活 profile 的属性名称-->
          <name>mavenVersion</name>
          <!--激活 profile 的属性值 -->
          <value>2.0.3</value>
        </property>
        <file>
          <!-- 如果指定的文件存在，则激活 profile -->
          <exists>${basedir}/file2.properties</exists>
          <!-- 如果指定的文件丢失，则激活 profile -->
          <missing>${basedir}/file1.properties</missing>
        </file>
      </activation>
      ...
    </profile>
  </profiles>
  ...
</settings>
```
~~当 `activation` 所有指定的条件都被满足的时，`profile`会被激活，同一次不需要所有的 `activation`~~。

(1) jdk

~~`activation` 在 `jdk` 元素中内置了一个Java版本检测~~。如果jdk版本号与给定的前缀相匹配，`profile` 将被激活。在上面的例子中，`1.5.0_06` 会匹配。`Maven 2.1` 也支持范围值。有关可支持的范围的更多详细信息，请参阅 [maven-enforcer-plugin](https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html)。

(2) os

`os` 元素可以定义如上显示的一些操作系统特定属性。有关 `OS` 值的更多细节，请参阅 [maven-enforcer-plugin]()。

(3) property

如果 `Maven` 检测到某一个对应`name = value`格式的属性（其值可以在POM中通过 `${name}` 引用）， `profile` 就会被激活。

(4) file

最后，通过检测给定的文件存在或丢失来激活 `profile`。`exists` 表示如果指定的文件存在，则激活 `profile`，`missing` 表示如果指定的文件不存在，则激活 `profile`。

`activation` 元素不是激活 `profile` 的唯一方法。`settings.xml` 文件的 `activeProfile` 元素可能包含 `profile` 的 `id`,从而激活 `profile`。它们也可以使用 `-P` 标志后逗号分隔的列表的命令行方式激活 `profile`（例如-P test）。

##### 2.5.2 Properties

##### 2.5.3 Repositories



##### 2.5.4 Plugin Repositories

仓库是两种主要类型工件的家。第一个工件作为其他工件的依赖。这是中央存储仓库中存储的大部分构件类型。另一种类型的工件就是插件。`Maven` 插件本身就是一种特殊类型的工件。正因为如此，插件仓库可能会从其他库中分离出来（尽管如此，我还没有听到有说服力的理由）。无论如何， `pluginRepositories` 元素块的结构与 `repositories` 元素相似。`pluginRepository` 元素分别指定 `Maven` 可以在哪里找到新插件的远程地址。

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <pluginRepositories>
    ...
  </pluginRepositories>
</settings>
```

#### 2.6 Active Profiles

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      https://maven.apache.org/xsd/settings-1.0.0.xsd">
  ...
  <activeProfiles>
   <!-- 要激活的 profile id -->
    <activeProfile>env-test</activeProfile>
  </activeProfiles>
</settings>
```

`settings.xml` 最后一部分是 `activeProfiles` 元素。包含一组 `activeProfile` 元素，每个 `activeProfile` 元素都含有一个 `profile id`（例如上述的 `env-test`）。`profile id`对应的 `profile` 将不受任何环境设置的影响而被激活。如果没有找到匹配的 `profile`，什么也不会发生。例如，上述的 `env-test `是一个 `activeProfile`，那么 `pom.xml` 文件中对应的 `profile`（或者带有对应 `id` 的 `profile.xml` 文件）将被激活，如果没有找到这个 `profile`，将正常运行。


原文: http://maven.apache.org/settings.html

参考： https://www.jianshu.com/p/110d897a5442

http://blog.csdn.net/u012152619/article/details/51485152
