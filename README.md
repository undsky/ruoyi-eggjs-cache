# ruoyi-eggjs-cache

> Egg plugin for cache

基于 [cache-manager](https://github.com/node-cache-manager/node-cache-manager) 的 Egg.js 缓存插件，支持多种缓存存储方式（内存、文件系统、Redis）。

| 公众号                                       | 微信交流群                                                      |
| -------------------------------------------- | --------------------------------------------------------------- |
| ![公众号](https://cdn.undsky.com/img/gh.jpg) | ![微信交流群](https://cdn.undsky.com/img/doudouqun.jpg?v=2.0.1) |

## 目录

- [特性](#特性)
- [安装](#安装)
- [支持的 egg 版本](#支持的-egg-版本)
- [开启插件](#开启插件)
- [配置](#配置)
- [使用方法](#使用方法)
- [使用场景](#使用场景)
- [缓存策略选择](#缓存策略选择)
- [环境配置](#环境配置)
- [完整示例](#完整示例)
- [常见问题](#常见问题)
- [性能优化建议](#性能优化建议)
- [完整项目](#完整项目)
- [请我喝杯咖啡](#请我喝杯咖啡)
- [联系方式](#联系方式)
- [贡献指南](#贡献指南)
- [License](#license)


## 特性

- ✅ 支持三种缓存存储：内存（Memory）、文件系统（FS）、Redis
- ✅ 灵活配置默认缓存方式
- ✅ 统一的缓存 API（get、set、del、wrap）
- ✅ 支持 TTL（过期时间）配置
- ✅ 开发环境自动短时缓存（1秒）
- ✅ 基于 cache-manager 生态，功能强大

## 安装

```bash
$ npm i ruoyi-eggjs-cache --save
```

## 支持的 egg 版本

| egg 3.x | egg 2.x | egg 1.x |
| ------- | ------- | ------- |
| 😁      | 😁      | ❌      |

## 开启插件

```js
// {app_root}/config/plugin.js
exports.cache = {
  enable: true,
  package: "ruoyi-eggjs-cache",
};
```

## 配置

### 基础配置

```js
// {app_root}/config/config.default.js
const path = require("path");

config.cache = {
  default: "redis", // 默认缓存方式：'memory' | 'fs' | 'redis'
  ttl: 600, // 缓存时长（秒），默认 10 分钟
  fs: {
    path: path.join(appInfo.baseDir, "cache"), // 文件缓存目录
    subdirs: false, // 是否使用子目录
    zip: false, // 是否压缩
  },
  redis: null, // Redis 配置，为 null 时不启用
};
```

### 启用 Redis 缓存

```js
// {app_root}/config/config.default.js
config.cache = {
  default: "redis",
  ttl: 600,
  fs: {
    path: path.join(appInfo.baseDir, "cache"),
    subdirs: false,
    zip: false,
  },
  redis: {
    host: "127.0.0.1",
    port: 6379,
    password: "",
    db: 0,
  },
};
```

### 配置参数说明

| 参数       | 类型    | 默认值 | 说明                            |
| ---------- | ------- | ------ | ------------------------------- |
| default    | String  | redis  | 默认缓存方式（memory/fs/redis） |
| ttl        | Number  | 600    | 缓存过期时间（秒）              |
| fs.path    | String  | -      | 文件缓存目录路径                |
| fs.subdirs | Boolean | false  | 是否使用子目录存储              |
| fs.zip     | Boolean | false  | 是否压缩缓存文件                |
| redis      | Object  | null   | Redis 连接配置                  |

## 使用方法

插件提供了四种缓存实例：

- `app.cache.memory` - 内存缓存
- `app.cache.fs` - 文件系统缓存
- `app.cache.redis` - Redis 缓存（需配置）
- `app.cache.default` - 默认缓存（根据配置自动选择）

### 基本操作

#### 1. 设置缓存 - set

```js
// 设置缓存，使用默认 TTL
await app.cache.default.set("key", "value");

// 设置缓存，自定义 TTL（秒）
await app.cache.default.set("key", { name: "张三" }, 300);

// 使用内存缓存
await app.cache.memory.set("key", "value");

// 使用文件缓存
await app.cache.fs.set("key", "value");

// 使用 Redis 缓存
await app.cache.redis.set("key", "value");
```

#### 2. 获取缓存 - get

```js
// 获取缓存
const value = await app.cache.default.get("key");

// 缓存不存在时返回 undefined
if (value === undefined) {
  console.log("缓存不存在或已过期");
}
```

#### 3. 删除缓存 - del

```js
// 删除指定的缓存
await app.cache.default.del("key");

// 删除多个缓存
await app.cache.default.del(["key1", "key2", "key3"]);
```

#### 4. 缓存包装器 - wrap（推荐）

`wrap` 是最常用的方法，它会自动处理缓存逻辑：

- 如果缓存存在，直接返回缓存值
- 如果缓存不存在，执行回调函数获取数据，并自动缓存

```js
// 基础用法
const result = await app.cache.default.wrap("key", async () => {
  // 这里的代码只在缓存不存在时执行
  return await app.mysql.select("SELECT * FROM users WHERE id = 1");
});

// 自定义 TTL
const result = await app.cache.default.wrap(
  "key",
  async () => {
    return await app.mysql.select("SELECT * FROM users");
  },
  { ttl: 300 },
); // 缓存 5 分钟
```

#### 5. 重置缓存 - reset

```js
// 清空所有缓存
await app.cache.default.reset();

// 清空内存缓存
await app.cache.memory.reset();

// 清空文件缓存
await app.cache.fs.reset();

// 清空 Redis 缓存
await app.cache.redis.reset();
```

## 使用场景

### 场景 1：数据库查询缓存

```js
// app/service/user.js
class UserService extends Service {
  async getUserById(userId) {
    const { app } = this;

    // 使用 wrap 自动缓存数据库查询结果
    return await app.cache.default.wrap(`user:${userId}`, async () => {
      const sql = app.mapper(
        "mapper/mysql/ruoyi/SysUserMapper.xml",
        "selectUserById",
        [userId],
      );
      return await app.mysql.select(sql);
    });
  }

  async updateUser(user) {
    const { app } = this;

    // 更新数据库
    const sql = app.mapper(
      "mapper/mysql/ruoyi/SysUserMapper.xml",
      "updateUser",
      [user.userId],
      user,
    );
    await app.mysql.update(sql);

    // 删除缓存，确保下次获取最新数据
    await app.cache.default.del(`user:${user.userId}`);
  }
}
```

### 场景 2：API 响应缓存

```js
// app/controller/user.js
class UserController extends Controller {
  async list() {
    const { ctx, app } = this;
    const params = ctx.request.body;

    // 根据查询参数生成缓存 key
    const cacheKey = `user:list:${JSON.stringify(params)}`;

    const users = await app.cache.default.wrap(
      cacheKey,
      async () => {
        const sql = app.mapper(
          "mapper/mysql/ruoyi/SysUserMapper.xml",
          "selectUserList",
          ctx.helper.page(params),
          params,
        );
        return await app.mysql.selects(sql);
      },
      { ttl: 60 },
    ); // 缓存 1 分钟

    ctx.body = {
      code: 200,
      data: users,
    };
  }
}
```

### 场景 3：配置缓存

```js
// app/service/config.js
class ConfigService extends Service {
  async getConfigByKey(configKey) {
    const { app } = this;

    // 配置信息长期缓存
    return await app.cache.default.wrap(
      `config:${configKey}`,
      async () => {
        const sql = app.mapper(
          "mapper/mysql/ruoyi/SysConfigMapper.xml",
          "selectConfigByKey",
          [configKey],
        );
        return await app.mysql.select(sql);
      },
      { ttl: 3600 },
    ); // 缓存 1 小时
  }

  async updateConfig(config) {
    const { app } = this;

    // 更新配置
    const sql = app.mapper(
      "mapper/mysql/ruoyi/SysConfigMapper.xml",
      "updateConfig",
      [config.configId],
      config,
    );
    await app.mysql.update(sql);

    // 清除配置缓存
    await app.cache.default.del(`config:${config.configKey}`);
  }
}
```

### 场景 4：使用不同的缓存存储

```js
// app/service/data.js
class DataService extends Service {
  async getFrequentData() {
    const { app } = this;

    // 频繁访问的数据使用内存缓存（速度最快）
    return await app.cache.memory.wrap("frequent:data", async () => {
      return await this.fetchFrequentData();
    });
  }

  async getLargeData() {
    const { app } = this;

    // 大数据使用文件系统缓存（节省内存）
    return await app.cache.fs.wrap(
      "large:data",
      async () => {
        return await this.fetchLargeData();
      },
      { ttl: 1800 },
    );
  }

  async getSharedData() {
    const { app } = this;

    // 多实例共享数据使用 Redis（支持分布式）
    return await app.cache.redis.wrap(
      "shared:data",
      async () => {
        return await this.fetchSharedData();
      },
      { ttl: 600 },
    );
  }
}
```

### 场景 5：批量缓存操作

```js
// app/service/dict.js
class DictService extends Service {
  async getDictDataByType(dictType) {
    const { app } = this;

    const cacheKey = `dict:${dictType}`;

    return await app.cache.default.wrap(cacheKey, async () => {
      const sql = app.mapper(
        "mapper/mysql/ruoyi/SysDictDataMapper.xml",
        "selectDictDataByType",
        [dictType],
      );
      return await app.mysql.selects(sql);
    });
  }

  async refreshDictCache() {
    const { app } = this;

    // 获取所有字典类型
    const dictTypes = await this.getAllDictTypes();

    // 批量删除字典缓存
    const cacheKeys = dictTypes.map((type) => `dict:${type.dictType}`);
    await app.cache.default.del(cacheKeys);

    // 或者直接清空所有缓存
    // await app.cache.default.reset();
  }
}
```

## 缓存策略选择

### 内存缓存（Memory）

**优点：**

- 速度最快
- 无需额外服务

**缺点：**

- 应用重启后丢失
- 占用应用内存
- 单实例，不共享

**适用场景：**

- 频繁访问的小数据
- 临时数据
- 单机部署

### 文件系统缓存（FS）

**优点：**

- 持久化存储
- 不占用应用内存
- 无需额外服务

**缺点：**

- 速度较慢
- 单实例，不共享
- 占用磁盘空间

**适用场景：**

- 大数据缓存
- 需要持久化的数据
- 单机部署

### Redis 缓存（推荐）

**优点：**

- 速度快
- 持久化存储
- 多实例共享
- 支持分布式

**缺点：**

- 需要 Redis 服务

**适用场景：**

- 生产环境
- 集群部署
- 需要共享的数据

## 环境配置

### 开发环境

开发环境下 TTL 自动设置为 1 秒，避免缓存影响开发调试：

```js
// config/config.local.js
config.cache = {
  default: "memory", // 开发环境使用内存缓存
  ttl: 600, // 实际会被覆盖为 1 秒
  redis: null,
};
```

### 生产环境

生产环境使用配置的 TTL 值：

```js
// config/config.prod.js
config.cache = {
  default: "redis",
  ttl: 600, // 10 分钟
  redis: {
    host: process.env.REDIS_HOST || "127.0.0.1",
    port: process.env.REDIS_PORT || 6379,
    password: process.env.REDIS_PASSWORD || "",
    db: 0,
  },
};
```

## 完整示例

```js
// app/service/article.js
const { Service } = require("egg");

class ArticleService extends Service {
  // 获取文章详情（带缓存）
  async getArticleById(articleId) {
    const { app } = this;
    const cacheKey = `article:${articleId}`;

    return await app.cache.default.wrap(
      cacheKey,
      async () => {
        const sql = app.mapper(
          "mapper/mysql/blog/ArticleMapper.xml",
          "selectArticleById",
          [articleId],
        );
        return await app.mysql.select(sql);
      },
      { ttl: 600 },
    ); // 缓存 10 分钟
  }

  // 获取文章列表（带缓存）
  async getArticleList(params) {
    const { app, ctx } = this;
    const cacheKey = `article:list:${JSON.stringify(params)}`;

    return await app.cache.default.wrap(
      cacheKey,
      async () => {
        const sql = app.mapper(
          "mapper/mysql/blog/ArticleMapper.xml",
          "selectArticleList",
          ctx.helper.page(params),
          params,
        );
        return await app.mysql.selects(sql);
      },
      { ttl: 60 },
    ); // 列表缓存 1 分钟
  }

  // 创建文章
  async createArticle(article) {
    const { app } = this;
    const sql = app.mapper(
      "mapper/mysql/blog/ArticleMapper.xml",
      "insertArticle",
      [],
      article,
    );
    return await app.mysql.insert(sql);
    // 新增不需要清除缓存
  }

  // 更新文章
  async updateArticle(article) {
    const { app } = this;
    const sql = app.mapper(
      "mapper/mysql/blog/ArticleMapper.xml",
      "updateArticle",
      [article.articleId],
      article,
    );
    const result = await app.mysql.update(sql);

    // 更新后清除相关缓存
    await app.cache.default.del(`article:${article.articleId}`);
    // 清除所有列表缓存（简单粗暴的方式）
    // 更优雅的方式是使用缓存 key 模式匹配（需要 Redis 支持）

    return result;
  }

  // 删除文章
  async deleteArticle(articleId) {
    const { app } = this;
    const sql = app.mapper(
      "mapper/mysql/blog/ArticleMapper.xml",
      "deleteArticle",
      [articleId],
    );
    const result = await app.mysql.del(sql);

    // 删除后清除缓存
    await app.cache.default.del(`article:${articleId}`);

    return result;
  }

  // 刷新所有文章缓存
  async refreshAllCache() {
    const { app } = this;
    await app.cache.default.reset();
  }
}

module.exports = ArticleService;
```

## 常见问题

### 1. 如何选择缓存存储方式？

- **开发环境**：使用 `memory`，简单快速
- **生产环境（单机）**：使用 `fs`，持久化且无需额外服务
- **生产环境（集群）**：使用 `redis`，支持多实例共享

### 2. 何时需要手动删除缓存？

在数据更新（UPDATE）或删除（DELETE）操作后，需要手动删除相关缓存，确保下次获取最新数据。

### 3. 缓存 key 如何设计？

建议使用有意义的前缀和参数：

```js
`user:${userId}``article:list:${category}:${page}``config:${configKey}`;
```

### 4. 如何调试缓存？

在开发环境下，TTL 被设置为 1 秒，缓存会快速过期，方便调试。或者临时禁用缓存：

```js
// 跳过缓存，直接查询
const result = await this.queryDatabase();
```

### 5. 缓存会自动过期吗？

会的。所有缓存都有 TTL（过期时间），到期后自动删除。

### 6. Redis 配置失败怎么办？

如果 Redis 配置为 null 或连接失败，当 `default` 设置为 `redis` 时，会自动降级到 `fs` 缓存。

## 性能优化建议

1. **合理设置 TTL**
   - 频繁变化的数据：短 TTL（1-5 分钟）
   - 稳定的数据：长 TTL（10-60 分钟）
   - 配置类数据：超长 TTL（1-24 小时）

2. **避免缓存雪崩**
   - 为不同类型的缓存设置不同的 TTL
   - 使用随机 TTL：`ttl: 300 + Math.random() * 60`

3. **缓存 key 设计**
   - 使用简短且有意义的 key
   - 避免过长的 key（影响性能）
   - 统一命名规范

4. **选择合适的存储**
   - 小数据、高频访问：Memory
   - 大数据、低频访问：FS
   - 共享数据、集群部署：Redis

## 完整项目

参考 [ruoyi-eggjs](https://github.com/undsky/ruoyi-eggjs) 项目查看完整使用示例。

## 请我喝杯咖啡

如果项目对你有帮助，可以请我喝杯咖啡 ☕️

<img src="https://cdn.undsky.com/img/weixin10.jpg" max-width="300" height="500" /> <img src="https://cdn.undsky.com/img/zhifubao10.jpg" max-width="300" height="500" />

## 联系方式

- 🌐 **网站**: [https://www.undsky.com](https://www.undsky.com)
- 📮 **Issues**: [提交问题或建议](https://github.com/undsky/ruoyi-eggjs-cache/issues)

## 贡献指南

欢迎提交 Issue 和 Pull Request！

如果这个项目对你有帮助，请给我们一个 ⭐️ Star 支持一下！

---

## License

[MIT](LICENSE)
