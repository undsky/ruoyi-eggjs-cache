# egg-psyduck-cache

> Egg.js 缓存插件，支持内存、文件、Redis 缓存

## 安装

```bash
$ npm i egg-psyduck-cache --save
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
  package: "egg-psyduck-cache",
};
```

## 配置

```js
// {app_root}/config/config.default.js
config.cache = {
  ttl: 600, // 缓存时长（秒）
  fs: {
    path: path.join(appInfo.baseDir, "cache"),
    subdirs: false,
    zip: false,
  },
  redis: null,
};
```

## 示例

```js
const result = await this.app.cache.fs.wrap("key", async () => {
  return await this.app.mysql.select("sql");
});
```

## License

[MIT](LICENSE)
