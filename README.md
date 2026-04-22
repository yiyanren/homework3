﻿# week05大作业
## AI 智能单词本
### 一.项目基本信息

- 学校:武汉科技大学
- 姓名:丁志伟
- 学号:202313407094

---

### 二.开发任务索引

 | 任务                                 | 说明                                                         |
  | :----------------------------------- | ------------------------------------------------------------ |
  | 用户注册                             | 密码哈希，禁止明文存储                                       |
  | 用户登录                             | JWT Token 签发,JWT Token 签发                                |
  | 智能查询                             | 优先读取数据库，未保存时根据`ai_provider `调用对应Al接口     |
  | 支持 DeepSeek 与通义千问两种 AI 模型 | 前端下拉选择                                                 |
  | AI 返回格式化 JSON                   | 释义 + 3 条英文例句                                          |
  | 手动保存单词到个人单词本             | 查词时不自动入库                                             |
  | 获取单词列表                         | 分页，前端提供分页器，后端接收page 和 page_size              |
  | 删除单词                             | deleted_at 字段软删除                                        |
  | 跨域处理                             | 开发环境使用 Vite Proxy，生产环境使用 Nginx 反向代理         |
  | 后端多阶段构建                       | 镜像精简                                                     |
  | 前端 Nginx 镜像                      | 自定义nginx.conf,作为外部访问入口                            |
  | 编排部署                             | Docker Compose 三服务编排（db / backend / frontend）,数据库通过挂载 `init.sql` 自动建表 |

  

---

### 三.项目简介

​		用户注册登录后，在查词页输入英文单词并选择 AI 模型，后端会先检查该用户是否已保存过此单词：若已保存则直接从数据库返回；若未保存则实时调用 AI 大模型，获取中文释义与 3 条英文例句并返回前端展示。用户确认后可点击「保存到单词本」手动持久化，单词本支持分页浏览与删除。

#### 架构图

```
浏览器
    │  HTTP :80
    ▼
┌─────────────────────────────────────────────────────┐
│                    Docker 内部网络                    │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  frontend 容器（Nginx :80）                   │    │
│  │  ├── 伺服 React 静态资源（Vite 构建产物）      │   │
│  │  └── /api/* ──proxy_pass──► backend:8080     │   │
│  └─────────────────────────────────────────────┘   │
│                        │                           │
│                        ▼                           │
│  ┌─────────────────────────────────────────────┐   │
│  │  backend 容器（Go + Gin :8080）               │   │
│  │  ├── POST /api/auth/register                 │   │
│  │  ├── POST /api/auth/login                    │   │
│  │  ├── GET  /api/words/query  [JWT]            │   │
│  │  ├── POST /api/words        [JWT]            │   │
│  │  ├── GET  /api/words        [JWT]            │   │
│  │  └── DELETE /api/words/:id [JWT]             │   │
│  │              │                               │   │
│  │    调用外部 AI 接口                            │   │
│  │    ├── DeepSeek API (HTTPS)                  │   │
│  │    └── 通义千问 API (HTTPS)                   │   │
│  └─────────────────────────────────────────────┘   │
│                        │ TCP :3306                  │
│                        ▼                           │
│  ┌─────────────────────────────────────────────┐   │
│  │  db 容器（MySQL 8.0）                         │   │
│  │  └── 挂载 init.sql 自动建表                   │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

### 四.运行指南

#### 1. 前置依赖

启动前请先确认本机已安装并启动：

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Docker Compose（Docker Desktop 一般已内置，可直接使用 `docker compose` 命令）

建议先执行一次下面命令，确认 Docker 正常运行：

```bash
docker --version
docker compose version
```

#### 2. 准备 AI API Key

本项目查词功能依赖 AI 服务，启动前请先准备至少一个可用的 API Key：

- **DeepSeek**：在 [platform.deepseek.com](https://platform.deepseek.com/) 获取
- **通义千问**：在 [dashscope.console.aliyun.com](https://dashscope.console.aliyun.com/) 获取

然后在项目根目录下创建或编辑 `backend/.env` 文件。可直接使用下面模板：

```env
PORT=8080
DB_DSN=root:rootpassword@tcp(127.0.0.1:3307)/wordbook?charset=utf8mb4&parseTime=True&loc=Local
JWT_SECRET=replace_with_a_random_secret
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
QIANWEN_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

说明：

- `DEEPSEEK_API_KEY` 和 `QIANWEN_API_KEY` 至少填写一个，否则查词接口无法正常调用对应模型。
- 如果只打算使用一种模型，另一项可以留空。
- `JWT_SECRET` 改成随机字符串。
- `DB_DSN` 这一项用于本地开发直连数据库；使用 Docker Compose 启动整套服务时，容器内会自动覆盖为 `db:3306`，这里保持模板内容即可。

#### 3. 一键启动项目

在项目根目录执行：

```bash
docker compose up -d
```

说明：

- 首次启动会自动拉取 MySQL 镜像并构建前后端镜像，通常需要几分钟。
- 如果修改过前端或后端代码，想强制重新构建，可执行：

```bash
docker compose up -d --build
```

可选检查命令：

```bash
docker compose ps
```

当看到 `db`、`backend`、`frontend` 三个服务都已启动后，就可以开始访问项目。

#### 4. 启动后如何访问

当前 `docker-compose.yml` 对外暴露的入口如下：

| 访问地址 | 用途 |
|------|------|
| [http://localhost](http://localhost) | 前端页面入口 |
| [http://localhost/api/auth/register](http://localhost/api/auth/register) | 后端注册接口 |
| [http://localhost/api/auth/login](http://localhost/api/auth/login) | 后端登录接口 |
| [http://localhost/api/words/query](http://localhost/api/words/query) | 后端查词接口（需先登录并携带 JWT） |
| `localhost:3307` | MySQL 对宿主机开放的端口，便于本地调试数据库 |

补充说明：

- 前端页面统一从 `http://localhost` 访问。
- 后端服务**没有直接映射宿主机 `8080` 端口**；在当前配置下，外部访问后端时请使用 `http://localhost/api/...`，由前端 Nginx 反向代理到后端容器。
- 如果刚启动就访问到 `502 Bad Gateway`，通常是后端还在等待 MySQL 就绪，稍等十几秒后刷新即可。

#### 5. 停止 / 清理

```bash
# 停止服务，但保留数据库数据
docker compose down

# 停止服务并删除数据库数据卷（会清空已有数据）
docker compose down -v
```

#### 开发环境本地运行

如需在本地修改代码并热更新，按以下步骤操作：

**前置：** 本地已安装 Go 1.21+、Node.js 22+

**1. 仅启动数据库容器：**

```bash
docker compose up db -d
```

**2. 启动后端（`backend/.env` 中 `DB_DSN` 须用 `localhost`）：**

```bash
cd backend
# 将 .env 中 DB_DSN 的主机名改为 localhost（生产部署时填 db）
go run main.go
```

**3. 启动前端开发服务器：**

```bash
cd frontend
npm install
npm run dev
```

访问 [http://localhost:5173](http://localhost:5173)，`/api` 请求自动代理到 `localhost:8080`。

---

### 五.核心技术实现

#### 1. 用户注册：密码哈希，禁止明文存储

注册逻辑位于 `backend/service/auth.go`。系统先检查用户名是否已存在，再使用 `bcrypt.GenerateFromPassword` 对密码做哈希，最终只保存 `PasswordHash`，不会把原始密码写入数据库。

```go
func (s *AuthService) Register(username, password string) error {
	var existing model.User
	if err := s.DB.Where("username = ?", username).First(&existing).Error; err == nil {
		return errors.New("username already exists")
	}

	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return err
	}

	user := model.User{
		Username:     username,
		PasswordHash: string(hash),
	}
	return s.DB.Create(&user).Error
}
```

对应的数据表设计也只保存 `password_hash`：

```sql
CREATE TABLE IF NOT EXISTS users (
    id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    username      VARCHAR(64)     NOT NULL,
    password_hash VARCHAR(255)    NOT NULL,
    ...
);
```

#### 2. 用户登录：JWT Token 签发与鉴权

登录时先通过 `bcrypt.CompareHashAndPassword` 校验密码，再签发带有 `user_id` 和过期时间的 JWT。后续所有需要登录的单词接口都挂在 JWT 中间件后面。

```go
func (s *AuthService) Login(username, password string) (string, error) {
	var user model.User
	if err := s.DB.Where("username = ?", username).First(&user).Error; err != nil {
		return "", errors.New("invalid username or password")
	}

	if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password)); err != nil {
		return "", errors.New("invalid username or password")
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"user_id": user.ID,
		"exp":     time.Now().Add(72 * time.Hour).Unix(),
	})

	return token.SignedString([]byte(config.C.JWTSecret))
}
```

路由层中，注册/登录接口公开，查词/保存/列表/删除接口则必须经过 JWT 中间件：

```go
apiGroup := r.Group("/api")
{
	auth := apiGroup.Group("/auth")
	{
		auth.POST("/register", authHandler.Register)
		auth.POST("/login", authHandler.Login)
	}

	words := apiGroup.Group("/words", middleware.JWTAuth())
	{
		words.GET("/query", wordHandler.QueryWord)
		words.POST("", wordHandler.SaveWord)
		words.GET("", wordHandler.ListWords)
		words.DELETE("/:id", wordHandler.DeleteWord)
	}
}
```

JWT 中间件从请求头读取 `Authorization: Bearer <token>`，验证成功后把 `user_id` 放入上下文，供业务层读取：

```go
func JWTAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		authHeader := c.GetHeader("Authorization")
		if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": 401, "message": "authorization header required"})
			return
		}

		tokenStr := strings.TrimPrefix(authHeader, "Bearer ")
		token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
			return []byte(config.C.JWTSecret), nil
		})
		if err != nil || !token.Valid {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": 401, "message": "invalid or expired token"})
			return
		}

		claims := token.Claims.(jwt.MapClaims)
		c.Set("user_id", uint(claims["user_id"].(float64)))
		c.Next()
	}
}
```

#### 3. 智能查询：优先查数据库，未命中再调用 AI

查词逻辑位于 `backend/service/word.go`。系统先按 `user_id + word` 查询数据库；若该单词已在用户个人单词本中，则直接返回缓存结果；如果未保存，再根据 `ai_provider` 调用外部 AI。

```go
func (s *WordService) QueryWord(userID uint, word, provider string) (*QueryResult, error) {
	var w model.Word
	err := s.DB.Where("user_id = ? AND word = ?", userID, word).First(&w).Error
	if err == nil {
		return &QueryResult{
			Word:       w.Word,
			Definition: w.Definition,
			Sentences:  []string(w.Sentences),
			AIProvider: w.AIProvider,
			FromCache:  true,
		}, nil
	}

	result, err := QueryAI(word, provider)
	if err != nil {
		return nil, err
	}
	return &QueryResult{
		Word:       word,
		Definition: result.Definition,
		Sentences:  result.Sentences,
		AIProvider: provider,
		FromCache:  false,
	}, nil
}
```

前端根据 `from_cache` 字段决定展示“来自数据库”还是展示 AI 模型名：

```tsx
<span className={`result-badge ${result.from_cache ? 'badge-cached' : 'badge-ai'}`}>
  {result.from_cache ? '已保存 · 来自数据库' : result.ai_provider}
</span>
```

#### 4. 支持 DeepSeek 与通义千问两种 AI 模型

前端页面通过下拉框让用户选择模型，后端再根据 `ai_provider` 分发到对应实现：

```tsx
<select value={provider} onChange={(e) => setProvider(e.target.value)}>
  <option value="deepseek">DeepSeek</option>
  <option value="qianwen">通义千问</option>
</select>
```

```go
func QueryAI(word, provider string) (*AIResult, error) {
	switch provider {
	case "deepseek":
		return queryDeepSeek(word)
	case "qianwen":
		return queryQianwen(word)
	default:
		return nil, fmt.Errorf("unsupported ai_provider: %s", provider)
	}
}
```

#### 5. AI 返回格式化 JSON：释义 + 3 条英文例句

为了让不同模型都输出统一结构，后端构造了严格的 `systemPrompt`，要求模型只返回 JSON；随后再次做 JSON 反序列化和长度校验，确保响应结构稳定。

```go
var systemPrompt = `You are an English vocabulary assistant. When given a word, respond ONLY with a valid JSON object (no markdown, no code fences) in this exact format:
{"definition":"<concise Chinese definition>","sentences":["<English example sentence 1>","<English example sentence 2>","<English example sentence 3>"]}`
```

```go
content := aiResp.Choices[0].Message.Content
var result AIResult
if err := json.Unmarshal([]byte(content), &result); err != nil {
	return nil, fmt.Errorf("AI returned invalid JSON: %w", err)
}

if len(result.Sentences) < 3 {
	return nil, errors.New("AI returned fewer than 3 example sentences")
}
```

对应返回结构如下：

```go
type AIResult struct {
	Definition string   `json:"definition"`
	Sentences  []string `json:"sentences"`
}
```

#### 6. 手动保存单词到个人单词本：查词不自动入库

系统设计为“查询”和“保存”分离。查词接口只返回结果，不直接写库；只有用户点击“保存到单词本”时，前端才会调用保存接口。

```tsx
const handleSave = async () => {
  if (!result) return
  await saveWord({
    word: result.word,
    definition: result.definition,
    sentences: result.sentences,
    ai_provider: result.ai_provider,
  })
}
```

后端保存逻辑也做了幂等处理：如果该单词已存在，则直接返回已有记录，避免重复插入。

```go
func (s *WordService) SaveWord(userID uint, input SaveWordInput) (*model.Word, error) {
	var existing model.Word
	if err := s.DB.Where("user_id = ? AND word = ?", userID, input.Word).First(&existing).Error; err == nil {
		return &existing, nil
	}

	w := model.Word{
		UserID:     userID,
		Word:       input.Word,
		Definition: input.Definition,
		Sentences:  model.StringSlice(input.Sentences),
		AIProvider: input.AIProvider,
	}
	if err := s.DB.Create(&w).Error; err != nil {
		return nil, err
	}
	return &w, nil
}
```

#### 7. 获取单词列表：后端分页 + 前端分页器

后端通过 `page` 和 `page_size` 计算偏移量，并返回总数、当前页、分页大小和数据项：

```go
func (s *WordService) ListWords(userID uint, page, pageSize int) (*WordListResult, error) {
	if page < 1 {
		page = 1
	}
	if pageSize < 1 || pageSize > 100 {
		pageSize = 10
	}
	offset := (page - 1) * pageSize

	var total int64
	s.DB.Model(&model.Word{}).Where("user_id = ?", userID).Count(&total)

	var words []model.Word
	if err := s.DB.Where("user_id = ?", userID).
		Order("created_at DESC").
		Limit(pageSize).Offset(offset).
		Find(&words).Error; err != nil {
		return nil, err
	}

	return &WordListResult{Total: total, Page: page, PageSize: pageSize, Items: words}, nil
}
```

前端固定 `PAGE_SIZE = 3`，并提供上一页/下一页按钮：

```tsx
const PAGE_SIZE = 3
const totalPages = data ? Math.max(1, Math.ceil(data.total / PAGE_SIZE)) : 1

const res = await listWords(p, PAGE_SIZE)
setData(res.data.data)
```

```tsx
<button disabled={page <= 1} onClick={() => setPage((p) => p - 1)}>上一页</button>
<span>第 {page} 页 / 共 {totalPages} 页（{data.total} 条）</span>
<button disabled={page >= totalPages} onClick={() => setPage((p) => p + 1)}>下一页</button>
```

#### 8. 删除单词：GORM 软删除

`Word` 模型包含 `gorm.DeletedAt` 字段，因此 `Delete` 默认执行软删除，只会写入 `deleted_at`，而不是物理删除数据。

```go
type Word struct {
	ID         uint           `gorm:"primaryKey;autoIncrement" json:"id"`
	UserID     uint           `gorm:"not null;index" json:"user_id"`
	Word       string         `gorm:"type:varchar(128);not null" json:"word"`
	Definition string         `gorm:"type:text;not null" json:"definition"`
	Sentences  StringSlice    `gorm:"type:json;not null" json:"sentences"`
	AIProvider string         `gorm:"column:ai_provider;type:varchar(32);not null" json:"ai_provider"`
	DeletedAt  gorm.DeletedAt `gorm:"index" json:"-"`
}
```

```go
func (s *WordService) DeleteWord(userID, wordID uint) error {
	result := s.DB.Where("id = ? AND user_id = ?", wordID, userID).Delete(&model.Word{})
	if result.RowsAffected == 0 {
		return errors.New("word not found or permission denied")
	}
	return result.Error
}
```

#### 9. 跨域处理：开发环境走 Vite Proxy，部署环境走 Nginx 反向代理

前端统一把请求发送到 `/api`，从而避免在代码里写死后端完整地址：

```ts
const http = axios.create({
  baseURL: '/api',
  timeout: 30000,
})
```

生产环境下，Nginx 将 `/api/` 转发到后端容器，前端与接口对浏览器来说属于同源访问：

```nginx
location /api/ {
    proxy_pass         http://backend:8080;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
}
```

#### 10. 后端多阶段构建：构建镜像与运行镜像分离

后端 Dockerfile 使用两阶段构建：第一阶段基于 Go 镜像编译二进制，第二阶段基于更轻量的 Alpine 仅复制可执行文件和必要运行时依赖，从而减小最终镜像体积。

```dockerfile
FROM golang:1.25.6-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM alpine:3.20
WORKDIR /app
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/server .
CMD ["./server"]
```

#### 11. 前端 Nginx 镜像：静态资源托管 + API 入口

前端同样采用多阶段构建：先用 Node 构建 Vite 产物，再放入 Nginx 镜像中对外提供静态资源，并让 Nginx 成为整个系统的访问入口。

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
CMD ["nginx", "-g", "daemon off;"]
```

#### 12. 编排部署：Docker Compose 三服务协作 + 自动建表

项目通过 `docker-compose.yml` 同时编排 `db`、`backend`、`frontend` 三个服务。其中 MySQL 通过健康检查保证就绪后再拉起后端；初始化 SQL 则通过挂载 `docs/init.sql` 自动执行。

```yaml
services:
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
      - ./docs/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-prootpassword"]

  backend:
    build:
      context: ./backend
    env_file:
      - ./backend/.env
    environment:
      DB_DSN: root:rootpassword@tcp(db:3306)/wordbook?charset=utf8mb4&parseTime=True&loc=Local
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend
```

数据库初始化脚本负责创建 `wordbook` 库以及 `users`、`words` 两张核心表：

```sql
CREATE DATABASE IF NOT EXISTS wordbook CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE wordbook;

CREATE TABLE IF NOT EXISTS users (...);
CREATE TABLE IF NOT EXISTS words (...);
```

---



### 六.问题说明


| 问题 | 原因 & 解决 |
|------|------------|
| 访问页面返回 502 Bad Gateway | backend 容器正在等待 MySQL 就绪，稍等 20 秒后刷新 |
| 查词返回 500 Internal Server Error | `backend/.env` 中对应 AI 的 API Key 未填写或填写有误 |
| 本地启动报错:Database connection failed (attempt 1/10): dial tcp: lookup db: no such host | `DB_DSN` 里的主机名是 `db`（Docker 容器名），本地直接 `go run` 时没有 Docker 网络，操作系统找不到 `db` 这个主机名，所以报 `no such host`。                                   解决方法：把 `.env` 里的 `db` 改成 `localhost`                                                                       本地已安装 MySQL，将 `docker-compose.yml` 中 db 的端口改为 `"3307:3306"`，同时更新 `.env` 中 DSN 端口 |
