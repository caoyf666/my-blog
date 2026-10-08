## 我的博客

这是一个基于 Hexo 和 Matery 主题的个人博客项目。

### 本地预览

```bash
npm install
npx hexo clean
npx hexo generate
npx hexo server
```

启动后访问：

```bash
http://localhost:4000
```

### 常用目录

```bash
source/_posts                 文章
_config.yml                   站点配置
themes/matery/_config.yml     主题配置
```

上线前请把 `_config.yml` 里的 `url`、部署仓库和主题里的社交链接改成自己的信息。
