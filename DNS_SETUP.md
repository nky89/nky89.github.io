# 自定义域名配置指南

## GitHub Pages 配置

本仓库已经配置好了自定义域名 `blog.nky89.com`。以下是需要在您的DNS服务商处进行的配置步骤。

## DNS 配置步骤

您需要在域名服务商（如阿里云、腾讯云、Cloudflare等）的DNS管理面板中添加以下记录：

### 方法一：使用 CNAME 记录（推荐）

在您的DNS管理面板中添加一条CNAME记录：

- **记录类型**: CNAME
- **主机记录**: blog
- **记录值**: nky89.github.io
- **TTL**: 600（或默认值）

### 方法二：使用 A 记录

如果您的DNS服务商不支持CNAME记录，可以使用A记录指向GitHub Pages的IP地址：

添加以下4条A记录：

- **记录类型**: A
- **主机记录**: blog
- **记录值**: 
  - 185.199.108.153
  - 185.199.109.153
  - 185.199.110.153
  - 185.199.111.153
- **TTL**: 600（或默认值）

同时添加以下4条AAAA记录（IPv6）：

- **记录类型**: AAAA
- **主机记录**: blog
- **记录值**:
  - 2606:50c0:8000::153
  - 2606:50c0:8001::153
  - 2606:50c0:8002::153
  - 2606:50c0:8003::153
- **TTL**: 600（或默认值）

## 验证配置

1. DNS配置生效通常需要几分钟到几小时
2. 可以使用以下命令验证DNS配置：
   ```bash
   # 验证CNAME记录
   nslookup blog.nky89.com
   
   # 或使用dig命令
   dig blog.nky89.com
   ```

3. DNS配置生效后，访问 https://blog.nky89.com 即可访问您的GitHub Pages站点

## GitHub Pages 设置

本仓库的配置文件已经包含：

1. `CNAME` 文件：包含自定义域名 `blog.nky89.com`
2. `_config.yml` 文件：已配置 `url` 和 `baseurl` 参数

在GitHub仓库的Settings > Pages中，您会看到：
- Custom domain: blog.nky89.com
- Enforce HTTPS: 建议开启（DNS配置生效后可以开启）

## 注意事项

1. DNS配置需要时间生效，请耐心等待
2. 首次配置可能需要24-48小时完全生效
3. 建议在DNS配置生效后，在GitHub Pages设置中开启 "Enforce HTTPS"
4. 确保CNAME文件不要被删除或修改

## 参考资料

- [GitHub Pages 自定义域名官方文档](https://docs.github.com/cn/pages/configuring-a-custom-domain-for-your-github-pages-site)
- [管理 GitHub Pages 站点的自定义域](https://docs.github.com/cn/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
