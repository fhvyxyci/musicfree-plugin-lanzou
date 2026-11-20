# MusicFree 插件 —— 蓝奏云

把蓝奏云网盘当音乐库用：全盘搜索、按文件夹当歌单/专辑、递归汇总「全部歌曲」、同名 lrc 歌词、读取文件内嵌标签。

> 插件本体 `lanzou.js` 单文件，**不依赖任何 npm 包**（只用 MusicFree 沙箱内置的 axios）。

**导入地址（粘进 MusicFree 的「从网络安装」即可）**

```
https://fhvyxyci.github.io/musicfree-plugin-lanzou/lanzou.js
```

**订阅源**（和阿里云盘插件一起装、一起更新；订阅源仓库里还放了[《用 AI 做同类网盘插件的提示词》](https://github.com/fhvyxyci/musicfree-plugins/blob/main/PROMPT.md)）

<https://github.com/fhvyxyci/musicfree-plugins>

## 目录

- [功能](#功能)
- [安装](#安装)
- [配置变量](#配置变量)
- [关于账号与会话](#关于账号与会话)
- [关于「索引」](#关于索引)
- [接口依据](#接口依据)
- [常见问题](#常见问题)
- [原理](#原理)
- [致谢](#致谢)
- [协议](#协议)

## 功能

| 入口 | 行为 |
| --- | --- |
| 搜索 · 单曲 | 全盘搜索音频文件，支持翻页 |
| 搜索 · 歌单 / 专辑 | 把网盘里的**文件夹**当歌单/专辑列出来（两者内容一致，只列含音频的文件夹） |
| 推荐歌单 | 「全部文件夹」标签，不输关键词直接浏览所有文件夹 |
| 排行榜 | 两组：「全部歌曲（含子目录）」递归汇总所有音频；「目录」按文件夹浏览 |
| 播放 | 取蓝奏云 CDN 直链 |
| 歌词 | 自动找同目录同名的 `.lrc` |
| 元数据 | 优先读文件内嵌标签（标题 / 歌手 / 专辑 / 封面 / 时长），读不到再从文件名解析 |

其它行为：

- 自动过滤非音频文件（`.lrc`、图片、压缩包等不会进结果）
- 文件名解析会先剥掉 `01 - `、`01.` 这类音轨号前缀，不会把 `02` 当成歌手
- 接口返回 `zt=4`（请求过频）时指数退避重试，重试用尽仍然报错，不静默吞掉

## 安装

**方式一：网络安装（推荐）**

在 MusicFree 的「插件管理 → 从网络安装」里填：

```
https://fhvyxyci.github.io/musicfree-plugin-lanzou/lanzou.js
```

**方式二：手动导入**

下载本仓库的 `lanzou.js`，在「插件管理 → 从本地安装」里选它。

**方式三：订阅（和阿里云盘插件一起装）**

订阅源地址：

```
https://fhvyxyci.github.io/musicfree-plugins/plugins.json
```

- **桌面版**：插件管理 → 订阅设置 → 添加，填上面这条 → 保存 → 回到插件管理点「更新订阅」
- **手机版（支持订阅源的新版本）**：插件管理 → 右上角三个点 → 订阅源管理 → 添加 → 返回 → 右上角三个点 → 同步订阅源

订阅后本插件和阿里云盘插件会一起装上、以后一起更新。订阅源仓库：https://github.com/fhvyxyci/musicfree-plugins

> GitHub Pages 在国内访问不稳定，打不开的话见下面「访问不了 github.io 怎么办」。

## 配置变量

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| 账号 | 是 | 蓝奏云登录手机号 / 邮箱 |
| 密码 | 是 | 蓝奏云登录密码 |
| Cookie（可选） | 否 | 登录提示「无法保存会话」时改填这里，见下 |
| 音乐目录 ID（可选） | 否 | 只扫某个目录时填它的 `folder_id`；留空表示整个网盘 |

> 账号密码和 Cookie 二选一即可：填了 Cookie 就直接用 Cookie，不会再去登录。

最小可用配置就是账号 + 密码。

### 怎么拿到「音乐目录 ID」

浏览器打开蓝奏云网盘，点进你的音乐文件夹，地址栏会是
`pc.woozooo.com/mydisk.php?item=files&action=index&folder_id=14278213` 这种形式，
`folder_id=` 后面那串数字就是。

### 什么时候需要填 Cookie

插件登录后会从响应头里读 `phpdisk_info` 这个会话 cookie，之后所有接口靠它鉴权。
**如果当前运行环境不把 `Set-Cookie` 暴露给插件**（部分手机端运行时是这样），登录会成功但拿不到会话，此时插件会明确报错并提示你改用 Cookie：

1. 浏览器登录蓝奏云
2. 按 F12 → Network → 点任意一个请求 → 找到请求头里的 `Cookie`
3. **把整条 Cookie 头的值都复制下来**填进插件的「Cookie」变量（不要只复制 `phpdisk_info` 那一段——浏览器手里还有 `acw_tc`、`acw_sc__v2` 等反爬用的 cookie，一起带上成功率更高）

**直接整行粘进去就行**，带不带 `Cookie:` 前缀、有没有换行、有没有引号都可以，插件会自动清理（一个多余的换行就会让请求头非法、服务端回 `login not`）。

插件会自动从中取出 `phpdisk_info` 作为会话，其余 cookie 原样带上。

## 关于账号与会话

蓝奏云没有 OAuth，也没有 refresh_token，它的「令牌」就是 `phpdisk_info` 这个会话 cookie（官方说法约 15 天有效）。

> 这也是 [OpenList 官方文档](https://docs.oplist.org/zh/guide/drivers/lanzou.html)给出的结论：蓝奏云挂载有三种模式（账户 / cookie / 链接），**官方推荐「账户」**，因为「cookie 模式需要手动更换 cookie」。本插件默认就走账户模式——每次启动自动登录、会话过期（`zt=9`）自动重登，不需要你手动维护 cookie。Cookie 只是给「登录被风控」的环境留的备用通道。所以这里做的是和刷新令牌等价的一套接力：

1. **运行期间**：会话存在模块内存里，同一次运行只登录一次
2. **重启之后**：会话落到 `musicfree/storage`（桌面版可用；移动端没有这个模块，静默降级，最多表现为每次启动重新登录一次）
3. **会话失效**：接口返回 `zt=9` 时用账号密码自动重新登录并重试原请求
4. **换账号自动作废旧缓存**：落盘时只存「配置指纹 + 会话」，指纹由账号+密码+目录算出；你在设置里改了账号，旧缓存直接不认

落盘键名是 `lanzou.session.v1` / `lanzou.index.v1`，都带插件前缀，不会和别的插件打架。

**安全性说明**：账号密码是明文存在 MusicFree 的插件变量里的（这是 MusicFree 用户变量的机制，插件改不了）。插件只会把它们发给 `up.woozooo.com` / `pc.woozooo.com`，**不经过任何第三方服务器**——本插件不依赖 OpenList 的公共令牌服务（蓝奏云也不在 `api.oplist.org` 的支持列表里）。介意明文存密码的话，可以只填 Cookie，代价是过期后要手动换。

## 关于「索引」

蓝奏云的接口**没有服务端搜索**，只有「按目录列文件」。所以「搜索单曲」「全部歌曲」「全部文件夹」都压在同一份递归索引上：

- 索引是**分片构建**的：每次方法调用最多花几秒扫一部分目录，游标存进 `musicfree/storage`，下次接着扫（沙箱单个方法只有 10 秒，不可能一次扫完大网盘）
- **第一次用会不全**：索引没建完时搜索结果只覆盖已扫过的目录；多翻几页、或者去排行榜「全部歌曲」往下拉，都会推动索引继续构建
- 建完之后索引会落盘，之后启动直接读，搜索是本地过滤，很快
- 目录结构变了（新增/删除文件夹）目前不会自动重建，需要重新触发一次完整扫描

## 接口依据

全部接口来自 **OpenList 的 `drivers/lanzou` 驱动**（2026-10 改版后的版本），没有猜测路径，也没有靠搜索找 API。

发布前用临时脚本对真实账号逐条验证过，结论：

| 项目 | 实测结果 |
| --- | --- |
| 登录 | `POST pc.woozooo.com/mlogin.php`（优先）或 `up.woozooo.com/mlogin.php`，表单 `task=3&uid=&pwd=`，返回 `{"zt":1,"info":"成功登录"}` |
| 反爬 | 登录页 / 分享页 / 直链页会返回 `acw_sc__v2` JS 挑战页，算法是 `unbox + 定长掩码异或`，纯字符串运算即可，无需 crypto |
| 鉴权 | 真正起作用的是 `phpdisk_info` cookie，**只带 PHPSESSID 会返回 `zt=9 login not`** |
| 会话号 | 客户端自带 `PHPSESSID` 时服务端会沿用，便于在读不到 Set-Cookie 的环境里保持会话可控 |
| 列目录 | `doupload.php?uid=&vei=` + `task=47&folder_id=` |
| 列文件 | 同上 `task=5&folder_id=&pg=`，**翻页要一直翻到返回空数组**，页大小并不固定（实测出现过一页 18 条） |
| 取直链 | `task=22&file_id=` → 分享页 → `/fn?` 内页（取 `wp_sign` / `ajaxdata` / `domain1`）→ `POST apifile` → `{dom,url}` → `GET {dom}/file/{url}` 302 → CDN 真实地址 |
| 直链特性 | CDN 支持 `Range`（返回 206），文件头是真的音频；播放不需要带任何请求头 |

## 常见问题

**第一次搜索要等几秒钟？**
蓝奏云没有服务端搜索，第一次搜索要顺手把目录扫一遍（插件单次调用最多用 7 秒，扫不完会落盘续跑）。网盘里文件不多的话通常一次就扫完，之后搜索是本地过滤，基本秒出。

**搜索出来只有几首？**
索引还没建完，见上面「关于『索引』」。多翻两页会继续建。

**提示「需要提取码」？**
分享时给文件设了提取码的文件暂不支持，插件会明确报这个错。把提取码去掉（或在网盘里重新分享一次）即可。

**播放失败 / 播到一半断了？**
蓝奏云直链有效期很短（十几分钟到半小时），插件缓存 5 分钟，过期会自动重取。如果是 CDN 直接 403，通常是蓝奏云那边限制了该文件的下载，换个时间再试。

**封面是空的？**
只有文件里真的内嵌了封面（ID3 的 APIC / FLAC 的 PICTURE / MP4 的 covr）才会显示。没有内嵌封面的文件，插件不会去网上找图。

**歌手全是「未知歌手」？**
文件名里没有 `歌手 - 歌名` 这种结构，文件里也没有内嵌标签。把文件命名成 `歌手 - 歌名.mp3` 就会正确解析。

**提示「Cookie 被截断了」？**
`phpdisk_info` 正常约 200 个字符。如果只有几个字符，说明复制时被截断了（常见于从会省略长值的界面复制）。重新在浏览器 F12 → Network → 任意请求 → Request Headers 里复制 Cookie 整行；或者干脆用「账号 + 密码」方式，完全不需要粘贴。

**搜索结果里出现「蓝奏云出错（…）」这一行？**
那是插件的错误提示行，不是歌曲。说明某一步失败了，把整行文字发给我就能定位（括号里是出错阶段，冒号后是原始错误）。

**提示「当前运行环境没有返回 Set-Cookie」？**
见上面「什么时候需要填 Cookie」。

**访问不了 github.io 怎么办？**
导入地址走的是 GitHub Pages，国内直连不稳。可以用镜像加速前缀替换域名，或者下载 `lanzou.js` 用本地安装。插件本身不需要联网校验，本地安装和网络安装功能完全一样。

**会被限流吗？**
短时间大量请求会返回 `zt=4`，插件会自动退避重试。索引是分片建的，正常使用不会触发。

## 已知限制

- **反爬挑战在个别客户端上过不去**。蓝奏云会对部分客户端下发 `acw_sc__v2` 挑战页，`acw_tc` 这类配套 cookie 由服务端随挑战页一起下发；如果某个运行环境（部分 React Native 版本）**不暴露 Set-Cookie**，插件就拿不到配套 cookie，纯 JS 沙箱里又没有浏览器环境去执行它那段混淆脚本，这条路就过不去。症状是「反爬校验重试超限」。
  - 插件已把登录改到**不弹挑战的 `pc.woozooo.com/mlogin.php`**，日常使用基本不会碰到这个问题
  - 万一碰到，报错末尾会直接写「本环境从未返回 Set-Cookie」，看到这句就说明是运行环境的限制，不是账号问题
  - 兜底办法：填 Cookie 模式（在电脑浏览器登录后复制整行），或改用电脑端 MusicFree
- 蓝奏云会更换 CDN 域名，个别时段某个域名会解析失败，稍等重试即可

## 原理

```
登录  pc.woozooo.com/mlogin.php（被挡时退回 up.woozooo.com）
  │  (acw_sc__v2 挑战 → 算出 cookie → 重发)
  ├─→ phpdisk_info 会话  ──→ 落盘 lanzou.session.v1
  │
  ├─ mydisk.php            取 uid / vei
  │
  ├─ doupload.php task=47  列文件夹 ─┐
  ├─ doupload.php task=5   列文件   ─┴─→ 递归索引 ──→ 落盘 lanzou.index.v1
  │                                         │
  │                                         ├─→ 搜索 / 全部歌曲 / 文件夹
  │
  └─ doupload.php task=22  取分享信息
        └─ 分享页 → /fn? 内页（wp_sign / ajaxdata / domain1）
              └─ POST apifile → {dom, url}
                    └─ GET {dom}/file/{url} → 302 → CDN 直链
                          ├─ Range 读文件头 → ID3 / FLAC / MP4 标签
                          └─ 同目录同名 .lrc → 歌词
```

## 致谢

- [MusicFree](https://github.com/maotoumao/MusicFree) —— 插件宿主，本插件按它和 [musicfree-skills](https://github.com/maotoumao/musicfree-skills) 的插件协议开发
- [OpenList](https://github.com/OpenListTeam/OpenList) —— `drivers/lanzou` 是本插件全部接口的依据，反爬算法与直链取参流程移植自该驱动
- [musicfree-plugin-aliyundrive](https://github.com/fhvyxyci/musicfree-plugin-aliyundrive) —— 同类插件的工程结构（分片遍历、会话接力、文件夹当歌单）参考了它

## 协议

本项目以 **GNU Affero General Public License v3.0 or later（AGPL-3.0-or-later）** 发布，完整文本见 [LICENSE](LICENSE)。

为什么是 AGPL-3.0 而不是更宽松的协议：

- 本插件的 `acw_sc__v2` 反爬算法（`unbox` 置换表、掩码异或）与下载直链的取参流程**移植自 OpenList 的 `drivers/lanzou`**，那部分属于 OpenList（AGPL-3.0）的代码，按协议必须沿用 AGPL-3.0
- 工程结构（分片遍历、`musicfree/storage` 会话接力）参考了同为 AGPL-3.0 的 `musicfree-plugin-aliyundrive`
- MusicFree 主程序是 AGPL-3.0，官方插件仓库 MusicFreePlugins 是 GPL-3.0；AGPL-3.0 与 GPL-3.0 可以结合，取上限用 AGPL-3.0 最稳妥

如果你要基于本项目二次开发并分发，请同样以 AGPL-3.0 开源。

## 免责声明

本项目仅用于学习和个人备份用途，与蓝奏云官方无关。请遵守蓝奏云的服务条款，不要用于任何商业用途或大规模抓取。使用本插件产生的一切后果由使用者自行承担。
