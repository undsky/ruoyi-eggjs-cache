# ruoyi-eggjs-cache

> Egg plugin for cache

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
