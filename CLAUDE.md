# CLAUDE.md

本文件为 Claude Code 在本仓库工作时的项目说明。**开始任何开发任务前，先读 `docs/00-术语与口径.md` 和 `docs/05-支付与分账方案.md`。**

## 1. 项目是什么

大学校园生活服务平台（校园版美团），运营方是大学。核心业务：点餐、到店自提、集配点配送、拼单、团购、跑腿。核心痛点是**商家分账**：校内食堂档口、校内商铺、校外商户、骑手、学校平台服务费之间合规、准确、及时地分钱。

当前阶段：**规划完成，准备进入开发**。代码尚未实现，`docs/` 是唯一事实来源。

## 2. 仓库结构（Git submodule）

```
meituan-shop-project/          # 主仓库：文档、CLAUDE.md、部署编排、子模块指针
├── docs/                      # 规划文档（PRD/架构/数据/支付/合规/路线图）
├── CLAUDE.md                  # Claude Code 项目说明
├── deploy/                    # （待建）docker-compose、nginx、prometheus
├── server/                    # submodule → meituan-shop-server
└── frontend/                  # submodule → meituan-shop-frontend
```

- **主仓库只放文档与跨端编排**，不放业务代码。
- 后端代码在 `server/` 子模块，前端代码在 `frontend/` 子模块。
- 子模块的提交必须先在子仓库提交并推送，再在主仓库提交指针变更。

### 克隆

```bash
git clone --recurse-submodules git@github.com:yuan50697105/meituan-shop-project.git
# 或 HTTPS
git clone --recurse-submodules https://github.com/yuan50697105/meituan-shop-project.git
```

已克隆后拉取子模块：

```bash
git submodule update --init --recursive
git pull --recurse-submodules
```

### 修改子模块

```bash
cd server
git checkout main
# 开发、提交、推送
git push origin main
cd ..
git add server
git commit -m "chore: bump server submodule"
git push
```

> 不要在子模块处于 detached HEAD 时直接提交；先 `git checkout main`。

## 3. 技术栈

| 层 | 选型 |
|---|---|
| 学生/商家/骑手端 | uni-app（Vue3 + TypeScript + Pinia + pnpm monorepo） |
| 管理后台 | Vue3 + Element Plus + ECharts |
| 后端 | Java 21 + Spring Boot 3.3 + MyBatis-Plus + Flyway |
| 数据库 | MySQL 8.0 |
| 缓存 | Redis 7 + Redisson |
| 存储 | MinIO / 阿里云 OSS |
| 部署 | Docker Compose + Nginx |
| 支付 | 微信支付服务商模式 + 特约商户 + 服务商分账 |

**不要擅自引入**：Spring Cloud、Nacos、Sentinel、SkyWalking、Elasticsearch、RocketMQ（首期）、Doris/ClickHouse、KMS、WORM。确需引入先在 `docs/03-技术架构.md` 记录理由。

## 4. 不可违反的约定

### 4.1 金额

- 所有金额用 `BIGINT` 存**分**；Java 计算用 `BigDecimal`，`HALF_UP`。
- 禁止 `double`/`float`；禁止 `DECIMAL` 存金额；禁止在接口返回元为单位的浮点数。
- 比例用 `INT` 存 bp（10000 = 100%）。

### 4.2 状态

- 订单、配送、支付、退款、分账状态**只能**使用 `docs/00-术语与口径.md` 的字典。
- 状态推进必须 CAS + 写状态流水；禁止跨状态跳转、禁止覆盖历史。

### 4.3 支付与分账（红线）

- 平台/学校**不得**统一收款再转付商户（二清）。
- 资金由微信支付直接清分；平台不设资金池、不做提现审批、不代付。
- 分账总额不得超过用户实付；平台补贴走对公月结，**不得进入分账**。
- 分账规则必须版本化，订单支付时冻结快照；历史订单只读快照。
- 改动资金相关代码前，必须同步更新 `docs/05-支付与分账方案.md` 并通过评审。

### 4.4 模块边界

- 跨模块只调用对方 `api` 包，**禁止直接查对方表**。
- 跨模块写操作用领域事件 + Outbox 最终一致。
- 购物车归 `ms-order`，库存归 `ms-product`，资金归 `ms-fund`。
- 用 ArchUnit 在 CI 中强制边界。

### 4.5 安全与隐私

- 手机号、身份证、银行卡加密存储；日志脱敏。
- 所有资源归属服务端校验，禁止只靠前端隐藏。
- 学生位置、消费记录按 PIPL 最小必要、单独同意。
- 高危操作（退款 >500 元、费率变更、分账重试）双人复核 + 审计日志。

## 5. 必读文档

| 文档 | 何时读 |
|---|---|
| `docs/00-术语与口径.md` | 每次开发前 |
| `docs/01-产品定位与商业模式.md` | 做产品/运营/定价 |
| `docs/02-PRD-功能规格.md` | 做功能 |
| `docs/03-技术架构.md` | 做架构/部署 |
| `docs/04-数据模型与核心流程.md` | 写 DDL/Mapper/服务 |
| `docs/05-支付与分账方案.md` | 任何资金相关改动 |
| `docs/06-接口规范.md` | 写接口/前端请求 |
| `docs/07-路线图与验收.md` | 排期/验收 |
| `docs/08-合规与风险登记册.md` | 上线/合规/应急 |
| `docs/09-测试与上线.md` | 测试/发布 |

## 6. 常用命令（代码落地后生效）

### 后端

```bash
cd server
mvn clean test
mvn spring-boot:run -pl campus-app
mvn -pl campus-app -am package -DskipTests
```

### 前端

```bash
cd frontend
pnpm install
pnpm dev:student      # 学生端
pnpm dev:merchant     # 商家端
pnpm dev:rider        # 骑手端
pnpm dev:admin        # 管理后台
pnpm build:student
```

### 部署

```bash
cd deploy
docker compose up -d
docker compose logs -f app
```

## 7. 提交规范

- `feat:` 新功能
- `fix:` 修复
- `docs:` 文档
- `refactor:` 重构
- `test:` 测试
- `chore:` 构建/依赖/配置
- 资金、状态机、权限相关提交必须在描述中说明影响范围。
- 禁止提交密钥、证书、真实学生数据、生产配置。

## 8. 测试要求

- 资金相关（金额、优惠分摊、退款、分账）单测覆盖率 ≥90%。
- 订单状态机、库存扣减、幂等逻辑必须有单测。
- 每个 P0 接口至少 1 个集成测试。
- CI 必须通过：单测、ArchUnit、OpenAPI 兼容性、依赖漏洞扫描。

## 9. 文档变更

- 任何口径变更先改 `docs/00`，再改下游文档。
- 状态码、金额单位、模块归属、单量基线不得在代码中另立标准。
- 新增「待定」项登记到 `docs/08-合规与风险登记册.md` 第 6 节。

## 10. 当前待办（开发前）

1. 校方拍板：运营主体、资金是否进学校账户、食堂档口定性、平台服务费、校园准入（见 `docs/08` 第 6 节）。
2. 微信支付服务商资质与首户进件。
3. 骑手 App 后台定位/推送技术验证。
4. 集配点场地意向。
5. 后端 `server` 与前端 `frontend` 骨架初始化。
