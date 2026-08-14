# RESTful 设计规范

## 1. 黄金法则
**用名词表示资源，用 HTTP 方法表示动作。**

- ❌ 错误：`/getUserById?id=1`（动词 + 参数）
- ✅ 正确：`GET /users/1`（名词 + HTTP 方法）

---

## 2. 资源用复数名词

| 资源 | 正确写法 | 错误写法 |
|------|----------|----------|
| 用户 | `/users` | `/user` |
| 订单 | `/orders` | `/order` |
| 文章 | `/posts` | `/post` |
| 商品 | `/products` | `/product` |
| 评论 | `/comments` | `/comment` |

**为什么用复数？** 因为 `GET /users` 返回的是一个用户**列表**，用复数更符合语义。

---

## 3. HTTP 方法映射

| 操作 | 方法 | URL | 说明 |
|------|------|-----|------|
| 获取所有资源 | GET | `/资源名` | 返回资源列表 |
| 获取单个资源 | GET | `/资源名/{id}` | 返回指定 ID 的资源 |
| 创建资源 | POST | `/资源名` | 新增一个资源 |
| 全量更新 | PUT | `/资源名/{id}` | 完全替换指定资源 |
| 部分更新 | PATCH | `/资源名/{id}` | 只修改指定资源的某些字段 |
| 删除资源 | DELETE | `/资源名/{id}` | 删除指定资源 |

---

## 4. 嵌套资源

如果一个资源从属于另一个资源，用嵌套表达：

| 场景 | URL |
|------|-----|
| 获取用户1的所有订单 | `GET /users/1/orders` |
| 获取用户1的订单2 | `GET /users/1/orders/2` |
| 为用户1创建新订单 | `POST /users/1/orders` |
| 获取文章1的所有评论 | `GET /posts/1/comments` |
| 在文章1下创建评论 | `POST /posts/1/comments` |
| 获取评论3的详情 | `GET /comments/3`（也可单独访问） |

---

## 5. 查询参数用于过滤/排序/分页

| 场景 | URL | 说明 |
|------|-----|------|
| 分页 | `/posts?page=2&size=10` | 第2页，每页10条 |
| 排序（降序） | `/posts?sort=-createdAt` | 按创建时间降序排列 |
| 排序（升序） | `/posts?sort=price` | 按价格升序排列 |
| 筛选 | `/posts?status=published` | 只返回已发布的文章 |
| 多条件筛选 | `/posts?author=3&status=draft` | 作者3的草稿文章 |
| 搜索 | `/posts?search=linux` | 标题/内容含“linux”的文章 |
| 字段选择 | `/users/1?fields=id,name,email` | 只返回指定字段 |

**原则**：参数用于**过滤、排序、分页**，不用于指定资源 ID。资源 ID 应该放在路径里（`/users/1`），而不是 `?id=1`。

---

## 6. 状态码在 RESTful 中的使用

| 操作 | 成功状态码 | 常见错误码 |
|------|-----------|-----------|
| GET | 200 OK | 404 Not Found |
| POST | 201 Created | 400 Bad Request |
| PUT | 200 OK | 404 Not Found / 400 Bad Request |
| PATCH | 200 OK | 404 Not Found / 400 Bad Request |
| DELETE | 204 No Content | 404 Not Found / 403 Forbidden |

**注意**：
- POST 成功时建议返回 **201 Created**，并带 `Location` 头指向新资源地址（如 `Location: /posts/101`）。
- DELETE 成功时建议返回 **204 No Content**（没有响应体），表示“删掉了，没别的内容返回”。

---

## 7. 实战场景示例

### 7.1 博客系统

| 操作 | 方法 | URL | 说明 |
|------|------|-----|------|
| 获取文章列表 | GET | `/posts` | 返回所有文章 |
| 获取单篇文章 | GET | `/posts/1` | 返回 ID=1 的文章 |
| 创建新文章 | POST | `/posts` | 新增一篇文章 |
| 全量更新文章 | PUT | `/posts/1` | 完全替换文章1 |
| 部分更新文章 | PATCH | `/posts/1` | 只改文章1的标题 |
| 删除文章 | DELETE | `/posts/1` | 删除文章1 |
| 获取文章1的所有评论 | GET | `/posts/1/comments` | 嵌套资源 |
| 获取评论3的详情 | GET | `/comments/3` | 也可单独访问 |
| 在文章1下创建评论 | POST | `/posts/1/comments` | 嵌套创建 |

### 7.2 电商系统

| 操作 | 方法 | URL | 说明 |
|------|------|-----|------|
| 获取商品列表 | GET | `/products` | 分页、筛选、排序 |
| 获取单个商品 | GET | `/products/101` | 返回商品101的详情 |
| 创建商品 | POST | `/products` | 新增商品（管理员） |
| 更新商品 | PUT | `/products/101` | 全量更新商品101 |
| 部分更新商品 | PATCH | `/products/101` | 只改商品101的价格 |
| 删除商品 | DELETE | `/products/101` | 删除商品101 |
| 获取用户5的订单列表 | GET | `/users/5/orders` | 嵌套资源 |
| 获取订单88的详情 | GET | `/orders/88` | 也可单独访问 |
| 取消订单 | PATCH | `/orders/88` | 修改订单状态为“已取消” |

### 7.3 用户系统

| 操作 | 方法 | URL | 说明 |
|------|------|-----|------|
| 用户注册 | POST | `/users` | 新建用户 |
| 用户登录 | POST | `/auth/login` | 登录接口（单独处理） |
| 获取用户信息 | GET | `/users/1` | 获取用户1的信息 |
| 更新用户信息 | PATCH | `/users/1` | 修改用户1的昵称/头像 |
| 修改密码 | PATCH | `/users/1/password` | 单独改密码 |
| 删除用户 | DELETE | `/users/1` | 注销账号 |
| 获取用户1的收藏列表 | GET | `/users/1/favorites` | 嵌套资源 |
| 用户1收藏商品5 | POST | `/users/1/favorites` | 嵌套创建 |
| 取消收藏 | DELETE | `/users/1/favorites/5` | 删除指定收藏 |

---

## 8. 好的 RESTful 设计 vs 烂设计（对比总结）

| 对比项 | 好设计（RESTful） | 烂设计（❌） |
|--------|-------------------|-------------|
| 获取用户 | `GET /users/1` | `GET /getUser?id=1` |
| 创建订单 | `POST /orders` | `POST /createOrder` |
| 删除文章 | `DELETE /posts/5` | `GET /deletePost?id=5` 或 `POST /removePost` |
| 更新商品 | `PATCH /products/3` | `POST /updateProduct` |
| 获取所有文章 | `GET /posts` | `GET /getAllPosts` |
| 资源命名 | 复数名词（`/users`） | 单数或动词（`/user`、`/getUsers`） |
| 操作方法 | 用 HTTP 方法（GET/POST/PUT/DELETE） | 在 URL 里写动词（`/get`、`/create`） |
| 参数传递 | 资源 ID 放在路径里（`/users/1`） | 资源 ID 放在查询参数里（`?id=1`） |

---

## 9. 改造练习（烂接口 → 好接口）

| 烂接口（❌） | 好接口（✅） |
|-------------|-------------|
| `GET /getUser?id=5` | `GET /users/5` |
| `POST /createProduct` | `POST /products` |
| `POST /deleteOrder` | `DELETE /orders/1`（需要指定具体 ID） |
| `GET /getAllArticles` | `GET /articles` |
| `POST /updateUserInfo` | `PATCH /users/1` |
| `GET /getUserOrders?uid=3` | `GET /users/3/orders` |
| `POST /removeProduct/5` | `DELETE /products/5` |

---

## 10. 一句话总结

**RESTful 的核心就是用 URL 表示“什么东西”（名词），用 HTTP 方法表示“怎么做”（动词）。** URL 里永远不出现动词。

## 11. API 版本管理

当接口需要升级且不兼容旧版本时，通过版本号区分。

| 方式 | 示例 | 说明 |
|------|------|------|
| URL 路径 | `/api/v1/users` → `/api/v2/users` | 最直观，推荐 |
| 查询参数 | `/users?version=1` | 不够清晰 |
| 请求头 | `Accept-Version: v2` | 隐蔽，不常用 |

**推荐做法**：在 URL 中显式标注版本号，如 `/api/v1/users` 和 `/api/v2/users` 并存，逐步淘汰旧版本。

---

## 12. 常见的“非 RESTful”但合理的接口

不是所有接口都必须严格遵循 RESTful，以下场景例外：

| 场景 | 接口示例 | 原因 |
|------|----------|------|
| 登录 | `POST /auth/login` | 不是资源操作，是动作 |
| 登出 | `POST /auth/logout` | 不是资源操作，是动作 |
| 刷新 Token | `POST /auth/refresh` | 不是资源操作 |
| 文件上传 | `POST /upload` | 文件本身不是 RESTful 资源 |
| 搜索 | `GET /search?q=keyword` | 跨资源搜索 |
| 统计 | `GET /stats/dashboard` | 计算类接口 |

**原则**：RESTful 是规范，不是法律。当它让接口变得别扭时，可以灵活变通。

---

## 13. RESTful 接口响应格式建议

一个好的 API 应该有统一的响应格式，便于前端处理。

### 成功响应
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "name": "张三"
    }
}

{
    "code": 0,
    "message": "success",
    "data": {
        "items": [
            { "id": 1, "name": "张三" },
            { "id": 2, "name": "李四" }
        ],
        "pagination": {
            "page": 1,
            "size": 10,
            "total": 100,
            "totalPages": 10
        }
    }
}
### 失败响应
{
    "code": 40001,
    "message": "用户邮箱格式不正确",
    "timestamp": "2026-08-14T10:00:00Z"
}