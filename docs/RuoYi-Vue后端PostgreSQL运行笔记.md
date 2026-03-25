# RuoYi-Vue 后端（PostgreSQL）运行笔记

本文档记录从克隆代码到成功运行 RuoYi-Vue 后端的完整步骤，数据库为 **PostgreSQL**（原版默认为 MySQL）。

---

## 一、环境准备

| 环境 | 要求 |
|------|------|
| JDK | 17（项目为 Spring Boot 4.x） |
| Maven | 3.x |
| PostgreSQL | 已安装并启动，默认端口 5432 |
| Redis | 已安装并启动，默认端口 6379（若依用于缓存/会话） |

---

## 二、克隆代码

```bash
git clone https://gitee.com/y_project/RuoYi-Vue
cd RuoYi-Vue
```

或直接在 Gitee 下载 ZIP 解压到本地（如 `D:\NihonLife\RuoYi-Vue`）。

---

## 三、数据库：创建库并执行脚本

### 3.1 创建数据库

在 PostgreSQL 中创建数据库（可用 pgAdmin、psql 或 IDEA Database 工具）：

```sql
CREATE DATABASE ry_vue
  WITH ENCODING 'UTF8'
       LC_COLLATE = 'zh_CN.UTF-8'
       LC_CTYPE = 'zh_CN.UTF-8'
       TEMPLATE = template0;
```

若已存在同名库，可跳过或先 `DROP DATABASE ry_vue;` 再创建。

### 3.2 执行建表与初始数据脚本

在 **ry_vue** 库上按顺序执行项目中的 SQL 文件（路径相对于项目根目录）：

1. **主库表结构与数据**  
   - 文件：`sql/ry_pg.sql`  
   - 说明：系统表、菜单、用户、角色等。

2. **定时任务表（可选）**  
   - 文件：`sql/quartz_pg.sql`  
   - 说明：若使用定时任务功能需要执行。

执行方式示例：

- **IDEA**：Database 工具连接 `ry_vue` → 右键库 → Run SQL Script → 选择 `ry_pg.sql`，再选 `quartz_pg.sql`。
- **psql**：  
  `psql -U postgres -d ry_vue -f sql/ry_pg.sql`  
  `psql -U postgres -d ry_vue -f sql/quartz_pg.sql`

---

## 四、后端配置修改（改为 PostgreSQL）

### 4.1 依赖：MySQL 驱动改为 PostgreSQL

**文件：** `ruoyi-admin/pom.xml`

**原样（MySQL）：**

```xml
<!-- Mysql驱动包 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

**改为：**

```xml
<!-- PostgreSQL驱动包 -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

父 POM 中如有 MySQL 的 `version` 或 `scope` 等，可删除或注释掉，只保留上述 PostgreSQL 依赖。

### 4.2 数据源配置

**文件：** `ruoyi-admin/src/main/resources/application-druid.yml`

修改为 PostgreSQL 连接信息，例如：

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driverClassName: org.postgresql.Driver
    druid:
      master:
        url: jdbc:postgresql://localhost:5432/ry_vue
        username: postgres
        password: 1234
      # slave 从库如不用可保持 enabled: false
```

请按实际环境修改：

- `url`：库名、端口、主机（非本机则改 `localhost`）。
- `username` / `password`：PostgreSQL 用户名与密码。

### 4.3 其他配置（按需）

- **application.yml**  
  - 端口：`server.port` 默认 8080，若冲突可改为 8081 等。  
  - Redis：`spring.data.redis` 下 host、port、password 与当前环境一致即可。

- **MyBatis 方言（若依已配置可忽略）**  
  若 `application.yml` 中有 PageHelper，方言应为 PostgreSQL，例如：  
  `helperDialect: postgresql`。

---

## 五、PostgreSQL 与 MySQL 语法差异（若报错再改）

若依部分 Mapper 原为 MySQL 写法，在 PostgreSQL 下可能报错，可按需修改：

| MySQL 写法 | PostgreSQL 写法 | 说明 |
|------------|-----------------|------|
| `sysdate()` | `CURRENT_TIMESTAMP` 或 `now()` | 当前时间 |
| `` `列名` `` | `"列名"` | 保留字或含特殊字符的列名 |
| `ifnull(a, b)` | `coalesce(a, b)` | 空值替换 |

常见出错位置：

- 登录日志：`ruoyi-system/.../mapper/system/SysLogininforMapper.xml` 中的 `sysdate()`。
- 菜单查询：`ruoyi-system/.../mapper/system/SysMenuMapper.xml` 中的 `` `query` `` 和 `ifnull`。

若启动、登录、菜单加载都正常，可暂不改；一旦报“函数不存在”或“语法错误”，再按上表在对应 XML 中替换。

---

## 六、编译与打包

在项目根目录执行（跳过测试，加快构建）：

```bash
cd D:\NihonLife\RuoYi-Vue
mvn clean package -DskipTests
```

成功时控制台末尾会有 `BUILD SUCCESS`，并生成：

- `ruoyi-admin/target/ruoyi-admin.jar`

---

## 七、运行前检查

1. **PostgreSQL**：服务已启动，`ry_vue` 已建库并执行过 `ry_pg.sql`（及可选 `quartz_pg.sql`）。
2. **Redis**：服务已启动，端口与 `application.yml` 一致。
3. **8080 端口**：若已被占用，要么先释放，要么在 `application.yml` 中改 `server.port`。

**检查 8080 是否被占用（Windows PowerShell）：**

```powershell
netstat -ano | findstr :8080
```

若有输出，记下最后一列 PID，终止进程：

```powershell
taskkill /PID <PID> /F
```

---

## 八、启动后端

### 方式一：IDEA 运行

1. 用 IDEA 打开项目根目录（含根 `pom.xml`）。
2. 若未识别为 Maven 项目：在根目录 `pom.xml` 上右键 → **Add as Maven Project**。
3. 找到主类：`ruoyi-admin` 模块下 `com.ruoyi.RuoYiApplication`。
4. 右键该类 → **Run 'RuoYiApplication'**（或先配置 Application 运行项再运行）。

### 方式二：命令行运行 JAR

```bash
cd D:\NihonLife\RuoYi-Vue
java -jar ruoyi-admin/target/ruoyi-admin.jar
```

---

## 九、确认启动成功

控制台出现类似输出即表示成功：

```
Started RuoYiApplication in x.xxx seconds
若依启动成功
```

可访问：

- 后端接口根地址：`http://localhost:8080`（若改了端口则替换为实际端口）
- Druid 监控：`http://localhost:8080/druid`（账号/密码见 application-druid.yml 中 statViewServlet 的 login-username / login-password）
- Swagger：`http://localhost:8080/swagger-ui.html`（若已启用）

---

## 十、简要检查清单

- [ ] JDK 17、Maven、PostgreSQL、Redis 已安装并可用  
- [ ] 已创建数据库 `ry_vue` 并执行 `sql/ry_pg.sql`（及可选 `sql/quartz_pg.sql`）  
- [ ] `ruoyi-admin/pom.xml` 使用 PostgreSQL 驱动  
- [ ] `application-druid.yml` 中 url、username、password 指向本机 `ry_vue`  
- [ ] Redis 地址/端口/密码与 `application.yml` 一致  
- [ ] 执行过 `mvn clean package -DskipTests` 且 BUILD SUCCESS  
- [ ] 8080 端口未被占用（或已改端口）  
- [ ] 若登录/菜单报错，按第五节修改 Mapper 中 MySQL 语法为 PostgreSQL  

按以上步骤操作即可在本地成功运行 RuoYi-Vue 后端（PostgreSQL 版）。
