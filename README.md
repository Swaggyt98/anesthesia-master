# 麻醉信息管理系统后端

本项目是麻醉信息管理系统的后端服务，面向患者围术期业务，提供患者信息、术前麻醉评估、术中记录、复苏记录、麻醉记录汇总，以及监护设备数据接入与波形查询等能力。

服务基于 Spring Boot 构建，对外提供 REST API，并通过 STOMP/WebSocket 接收设备数据；业务数据和监护时序数据存储在 PostgreSQL 中。

> 本项目涉及医疗业务数据。部署和二次开发时，请遵守所在地区的隐私保护、数据安全和医疗软件相关要求，禁止将真实患者数据、数据库备份、证书私钥或生产环境凭据提交到代码仓库。

## 项目背景

麻醉业务横跨术前、术中和术后多个阶段，数据来源包括医护人员录入、麻醉工作站及监护设备。传统的分散记录方式不利于信息衔接、过程追溯和数据汇总。

本项目用于统一承载麻醉业务数据，主要目标包括：

- 串联患者从术前评估到术后复苏的业务记录。
- 接收监护设备产生的生命体征参数和波形数据。
- 为前端应用提供统一的查询、录入和汇总接口。
- 支持麻醉记录、复苏事件和医护签名等信息的结构化保存。

## 技术栈

| 分类 | 技术 |
| --- | --- |
| 开发语言 | Java 21 |
| 应用框架 | Spring Boot 3.4.11 |
| Web | Spring MVC、Spring WebSocket |
| 安全认证 | Spring Security、JWT（JJWT） |
| 数据访问 | MyBatis、Spring Data JPA |
| 数据库 | PostgreSQL，可选 TimescaleDB 扩展 |
| 参数校验 | Jakarta Validation |
| 构建工具 | Maven、Maven Wrapper |
| 测试 | JUnit 5、Mockito、Spring Boot Test |
| 部署 | Docker，多阶段镜像构建 |

## 功能特性

### 围术期业务

- 患者基本信息查询。
- 手术及诊疗记录创建。
- 术前麻醉评估（PAA）查询和保存。
- 术中麻醉区域记录、麻醉医师信息和签名管理。
- 复苏评估、出复苏室记录及复苏事件记录（RER）。
- 麻醉记录汇总（ARS）及患者列表查询。
- 医护人员信息查询。

### 设备与监护数据

- 根据设备编号接收监护数据。
- 通过 STOMP/WebSocket 订阅设备数据流。
- 根据设备绑定关系关联诊疗记录。
- 保存 ECG、SpO2、呼吸等波形数据。
- 保存心率、血压、血氧、体温、呼吸频率等参数数据。
- 按诊疗记录查询波形及参数序列。

### 安全与工程能力

- 提供用户注册、登录和 JWT 认证基础能力。
- 支持通过配置启用或关闭认证。
- 使用 XML Mapper 管理复杂 SQL 和报表查询。
- 支持本地运行和 Docker 镜像部署。
- 使用非 root 用户运行生产容器。

## 项目结构

```text
.
├── src
│   ├── main
│   │   ├── java/com/medical
│   │   │   ├── config       # Web、WebSocket 和认证配置
│   │   │   ├── controller   # REST API 控制器
│   │   │   ├── mapper       # MyBatis Mapper 接口
│   │   │   ├── pojo         # 实体、DTO 和请求对象
│   │   │   ├── repository   # JPA Repository
│   │   │   ├── security     # JWT 和 Spring Security
│   │   │   ├── service      # 业务服务
│   │   │   ├── stomp        # 设备数据订阅与消费
│   │   │   └── utils        # 通用工具
│   │   └── resources
│   │       ├── mapper        # MyBatis XML Mapper
│   │       ├── application.yml
│   │       └── application-docker.yml
│   └── test                  # 单元测试
├── sql                       # 时序数据表初始化脚本
├── Dockerfile
├── Dockerfile.dev
├── pom.xml
├── mvnw
└── mvnw.cmd
```

## 环境要求

- JDK 21
- PostgreSQL
- Maven 3.9+，或直接使用仓库中的 Maven Wrapper
- 可选：TimescaleDB，用于波形时序数据的分区和压缩
- 可选：Docker 及 Docker Compose

应用默认监听 `8080` 端口。

## 使用说明

### 1. 获取代码

```bash
git clone https://github.com/<your-account>/anesthesia-backend.git
cd anesthesia-backend
```

### 2. 准备数据库

先创建 PostgreSQL 数据库，并导入项目运行所需的患者、诊疗、评估、术中记录和复苏记录等业务表结构。

仓库中的 `sql/create_timeseries_tables.sql` 只负责创建波形、生命体征参数及参数类型相关表，不包含全部业务表。执行该脚本前请确认连接的是正确数据库；脚本可能备份并重建已有的波形表。

```bash
psql -h localhost -U postgres -d anesthesia -f sql/create_timeseries_tables.sql
```

如果数据库安装了 TimescaleDB，脚本会尝试将波形表转换为超表并配置压缩策略；否则使用标准 PostgreSQL 表和索引。

### 3. 配置环境变量

不要把真实密码或密钥写入 Git。推荐通过环境变量覆盖配置：

| 环境变量 | 说明 | 示例 |
| --- | --- | --- |
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC 地址 | `jdbc:postgresql://localhost:5432/anesthesia` |
| `SPRING_DATASOURCE_USERNAME` | 数据库用户名 | `postgres` |
| `SPRING_DATASOURCE_PASSWORD` | 数据库密码 | 请使用本地或部署环境中的真实值 |
| `JWT_SECRET` | JWT 签名密钥 | 使用足够长的随机密钥 |
| `STOMP_SERVER_URL` | 设备数据 WebSocket 地址 | `ws://localhost:8080/ws` |
| `STOMP_TOPIC` | STOMP 订阅主题 | `/data/sub/<device-id>` |
| `STOMP_DEVICE_ID` | 默认设备编号 | `<device-id>` |
| `DEVICE_BINDING_URL` | 设备绑定服务地址 | `http://localhost:8080/device/binding/device/` |

PowerShell 示例：

```powershell
$env:SPRING_DATASOURCE_URL = "jdbc:postgresql://localhost:5432/anesthesia"
$env:SPRING_DATASOURCE_USERNAME = "postgres"
$env:SPRING_DATASOURCE_PASSWORD = "<your-database-password>"
$env:JWT_SECRET = "<your-random-jwt-secret>"
```

生产环境应使用密钥管理服务、容器 Secret 或 CI/CD Secret 管理凭据。

### 4. 本地启动

Windows：

```powershell
.\mvnw.cmd spring-boot:run
```

Linux 或 macOS：

```bash
./mvnw spring-boot:run
```

启动后，服务地址为：

```text
http://localhost:8080
```

### 5. 运行测试

Windows：

```powershell
.\mvnw.cmd test
```

Linux 或 macOS：

```bash
./mvnw test
```

### 6. 构建可执行 JAR

```bash
./mvnw clean package
java -jar target/project-0.0.1-SNAPSHOT.jar
```

Windows 可将 `./mvnw` 替换为 `.\mvnw.cmd`。

### 7. Docker 构建与运行

构建镜像：

```bash
docker build -t anesthesia-backend:latest .
```

运行镜像：

```bash
docker run --rm \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=docker \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/anesthesia \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD='<your-database-password>' \
  -e JWT_SECRET='<your-random-jwt-secret>' \
  anesthesia-backend:latest
```

Linux 环境访问宿主机数据库时，需要根据网络模式调整数据库主机名。

## 主要 API

| 模块 | 基础路径 | 说明 |
| --- | --- | --- |
| 用户认证 | `/auth` | 注册、登录 |
| 患者 | `/patients` | 患者信息查询 |
| 手术记录 | `/surgery` | 创建手术及诊疗记录 |
| 术前评估 | `/paa` | 查询、保存术前麻醉评估 |
| 术中记录 | `/surgeryArea` | 术中记录、麻醉医师和签名 |
| 复苏记录 | `/recovery` | 复苏评估和出室记录 |
| 复苏事件 | `/rer` | 保存复苏事件记录 |
| 麻醉记录汇总 | `/ARS` | 患者列表和麻醉记录汇总 |
| 波形数据 | `/waveform` | 查询诊疗相关波形数据 |
| 设备数据 | `/data` | 接收设备数据 |
| 医护人员 | `/staff` | 查询医护人员信息 |

具体请求字段和响应结构以 `controller`、`pojo/DTO` 及 Mapper 定义为准。

## 配置与安全建议

- 当前认证功能可通过 `auth.enabled` 配置；生产环境应启用认证并验证接口访问策略。
- 必须为生产环境生成独立的高强度 JWT 密钥。
- 不要提交 `.env`、数据库备份、日志、患者数据、证书或私钥。
- 限制数据库和设备绑定服务的网络访问范围。
- 正式部署前应补充数据库迁移方案、接口文档和覆盖关键业务流程的集成测试。
- 公开仓库前请确认代码、测试数据和提交历史中均不包含个人信息或内部网络信息。

## 相关文档

- `ARCHITECTURE.md`：系统架构和业务模块说明。
- `sql/create_timeseries_tables.sql`：波形及时序参数表初始化脚本。
- `Dockerfile`：生产镜像构建配置。
- `Dockerfile.dev`：容器开发环境配置。

## 许可证

本项目暂未声明开源许可证。在添加许可证之前，默认保留所有权利。如果计划公开开源，请根据项目归属和使用要求选择合适的许可证。
