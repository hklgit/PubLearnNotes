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

#### 2.2 Plugin Groups

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

#### 2.3 Servers

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

这是与Maven尝试连接到的存储库/镜像的id元素相匹配的服务器的标识（不是用户登录的标识）。

(2) username, password

(3) privateKey, passphrase

(4) filePermissions, directoryPermissions


#### 2.4 镜像

#### 2.5 代理

#### 2.5 Profiles

#### 2.6 Active Profiles


### 3. Example
```
<?xml version="1.0"?>
<settings xmlns="http://maven.apache.org/POM/4.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                            http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>snapshots</id>
            <username>snapshots</username>
            <password>g2Dki6XCfhi48Bnj</password>
            <filePermissions>664</filePermissions>
            <directoryPermissions>775</directoryPermissions>
        </server>
        <server>
            <id>releases</id>
            <username>jifeng.si</username>
            <password>Qunar.2614</password>
            <filePermissions>664</filePermissions>
            <directoryPermissions>775</directoryPermissions>
        </server>
    </servers>

    <profiles>
        <profile>
            <id>QunarNexus</id>
            <repositories>
                <repository>
                    <id>QunarNexus</id>
                    <url>http://svn.corp.qunar.com:8081/nexus/content/groups/public</url>
                    <releases>
                        <enabled>true</enabled>
                        <!-- always , daily (default), interval:X (where X is an integer in minutes) or never.-->
                        <updatePolicy>daily</updatePolicy>
                        <checksumPolicy>warn</checksumPolicy>
                    </releases>
                    <snapshots>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>QunarNexus</id>
                    <url>http://svn.corp.qunar.com:8081/nexus/content/groups/public</url>
                    <releases>
                        <enabled>true</enabled>
                        <checksumPolicy>warn</checksumPolicy>
                    </releases>
                    <snapshots>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>QunarNexus</activeProfile>
    </activeProfiles>

</settings>
```



原文: http://maven.apache.org/settings.html
