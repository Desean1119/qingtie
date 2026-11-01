# 部署 · 请帖上公网

页面是单文件 `qingtie/index.html`，零外链、无构建、无依赖。任何一种静态落点都能直接跑。

## 落点选择（按微信可达性排序）

| 落点 | 微信内可达 | 需要的凭据 |
|---|---|---|
| 已备案域名 + 任意静态服务（nginx / COS / OSS 静态站） | 稳定 | 服务器或云桶权限 |
| GitHub Pages（`<user>.github.io/qingtie/`） | **不稳**，国内间歇被拦 | 一个 classic PAT |
| 本机 `file://` 双击打开 | 不可分享 | 无 |

`github.io` 在中国大陆走微信内置浏览器时经常被拦。所以本页同时保留了图片链路：
`30-cover.png` 与 `31-invite.png` 可以直接发微信，她回时间后再生成 `34-final.png`。
链接是加分项，图片是兜底，两条腿走路。

## 路径 A · GitHub Pages（当前选定）

### 1. 造 Token

GitHub → 头像 → **Settings** → 左侧最底 **Developer settings** →
**Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**

- Note：`cindy-invitation`
- Expiration：`30 days`
- Scopes：只勾 **`repo`**（全选子项）

> 用 fine-grained token 不行：新仓库还不存在，无法把授权范围限定到它，
> 而 "All repositories" 又要账号管理员权限。classic + `repo` 是这条路的最小授权。
>
> 若账号在开了 SAML SSO 的组织里，生成后还需对该组织 Authorize。

### 2. 填进 Cindy

主界面侧边栏 **插件** → **GitHub** → 备用 **Personal Access Token** 粘贴保存。

本机没装 `gh` CLI，也没有 git / node，所以插件只会走 PAT 这条路。
填好后回一句「好了」，下面这些我一次做完，不用你再动手：

1. `create_repository` 建 `qingtie`（public + auto_init，拿到 `main` 分支）
2. GraphQL `createPagesSite` 开 Pages（插件的 `create_repository` 不带 `has_pages` 参数，
   REST 建仓库时开不了，只能建完再补这一步）
3. `create_or_update_file_contents` 传 `index.html`（base64，约 40 KB，单次提交即可）
4. 轮询 `repository { pagesSite { status url } }` 直到 `built`
5. 把最终地址发你，并用它跑一遍 `test.ps1` 的三条链路（flow / settle / restore）

### 3. 拿到地址后

发她的消息里就一句话加一条链接：

> 范老师，小任定了个薄酌，地址容小的我安排。
> 时辰请您自己挑，点开这个帖儿就行：<链接>

链接形如 `https://<user>.github.io/qingtie/`。她落帖后页面会生成一条自带答案的链接
（`?mode=final&d=2026-11-08&t=19:30&memo=…`），她复制发回，你点开直接就是定帖 —— 这一步
不需要后端，纯静态站就能闭环，也是"双方定完再通知一次"的实现方式。

## 路径 B · 自己的域名 / 云桶

把 `qingtie/index.html` 传成 `index.html` 即可，无需其它文件。若走 OSS/COS 静态网站托管，
记得把首页设为 `index.html`，并给响应加 `Content-Type: text/html; charset=utf-8`。

绑定自有域名（路径 A 也适用）：仓库根放一个 `CNAME` 文件写域名，再在 DNS 指过去。
这样能一并绕开 `github.io` 被拦的问题 —— 域名是你自己的，微信拦截只看域名不看内容。

## 改内容

只动 `index.html` 顶部 `CONFIG` 一块：

```js
guest:      "范老师",              // 抬头
host:       "服务员　小任",         // 落款
seal:       "小任<br>之印",        // 印章，<br> 控制换行
venue:      { name:"", addr:"", tel:"", mapUrl:"" },   // 席设，四项填齐即上帖
venuePh:    "交给小的我来安排",      // 席设留空时印这句
rangeFrom:  1, rangeDays: 45,      // 可择范围：明日起 45 天
slots:      [ {label:"午宴",time:"11:30"}, … ],        // 时段雅称
notifyWebhook: "",                  // 填飞书/企群机器人后，她一落帖你就收到
storeKey:   "qingtie.fan.v2"        // 换人做帖时改掉，免得读到上一位的记录
```

`mapUrl` 填高德/腾讯地图的 marker 链接，席设那一栏会自动多出「地图导航」按钮。

## 本地验证

```powershell
.\qingtie\test.ps1     # 起临时 HTTP 服务，跑 flow / settle / restore 三条链路
.\qingtie\build.ps1    # 出四态成图到 qingtie\3*.png（无头 Edge，2x 缩放）
```

`test.ps1` 依赖 `start-server.ps1`（.NET HttpListener，127.0.0.1:8791）。
两个脚本必须保持纯 ASCII：Windows PowerShell 5.1 按 ANSI 读无 BOM 的 `.ps1`，
中文注释会把解析器切坏。
