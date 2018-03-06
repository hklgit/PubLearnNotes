Nexus Repository Manager可以使用轻量级目录访问协议（LDAP）通过提供LDAP支持的外部系统（如Microsoft Exchange / Active Directory，OpenLDAP，ApacheDS等）进行身份验证。

配置LDAP可以通过几个简单的步骤来实现：
- 启用 `LDAP Authentication Realm`
- 使用连接和用户/组映射详细信息创建LDAP服务器配置
- 创建外部角色映射以使LDAP角色适应存储库管理器的特定用法

### 1. 启用 LDAP Authentication Realm

如下图 `安全领域管理` 所示，按照以下步骤激活 `LDAP Realm`：
- 跳转到 `Realms` 管理部分
- 选择 `LDAP Realm` 并将其添加到右侧的 `Active realms` 列表中
- 确保 `LDAP Realm` 位于列表中 `Local Authenticating Realm` 的下方
- 点击保存

![](https://help.sonatype.com/download/attachments/5411788/realms.png?version=2&modificationDate=1510763355615&api=v2)

最佳实践是激活 `Local Authenticating Realm` 以及 `Local Authorizing Realm`，以便即使 `LDAP 认证` 脱机或不可用，匿名，管理员和此 `realm` 中配置的其他用户也可以使用存储库管理器。如果在 `Local Authenticating Realm` 找不到任何用户帐户，都将通过 `LDAP` 身份验证。

### 2. LDAP连接和身份验证

如下图中所示的 `LDAP` 功能视图可通过管理菜单中的 `Security` 中的 `LDAP` 获得。

![](https://help.sonatype.com/download/attachments/5411804/ldap-feature.png?version=1&modificationDate=1508913946541&api=v2)

`Order` 确定了在对用户进行身份验证时，存储库管理器以何种顺序连接到 `LDAP` 服务器。`Name` 和 `URL` 列标识一个配置，并单击单个行时可访问 `Connection` ， `User` 和 `group` 配置部分。

`Create connection` 按钮可用于创建新的 `LDAP` 服务器配置。可以创建多个配置，并且可以在列表中访问。`Change order` 按钮可用于更改存储库管理器在弹出对话框中查询 `LDAP` 服务器的顺序。缓存成功的身份验证，以便后续登录每次都不用对 `LDAP` 服务器进行新的查询。`Clear cache` 按钮可用于删除这些缓存的认证。

> 备注

> 请联系你的LDAP服务器的管理员以确定正确的参数，因为它们在不同的LDAP服务器供应商，版本和管理员执行的各种配置之间有所不同。

以下参数允许您创建LDAP连接：
