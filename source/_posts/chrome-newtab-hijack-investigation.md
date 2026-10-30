---
title: Chrome 每次打开都跳到 aitab.360.cn，我顺着时间线把源头挖出来了
date: 2026-10-30
tags: [Chrome, 浏览器, 排障]
---

# Chrome 每次打开都跳到 aitab.360.cn，我顺着时间线把源头挖出来了

今天早上开电脑，点开 Chrome，又落到那个页面：

```
https://aitab.360.cn/newtab.html?djsource=nO6TdR&search=3037095
```

关掉重开，还是它。新建标签页，也是它。但进设置一看，主页是空的，启动时选的是"新标签页"，搜索引擎看着也是正常的。这种"设置里一切正常但行为不对"的情况，基本可以断定不是我改的，是有人动过配置。

## 先按常规套路排一遍

我的第一反应是老几样：快捷方式后面挂 URL、hosts、计划任务、启动项。

结果全是干净的。桌面和任务栏的 `Google Chrome.lnk` 参数只有 `--user-data-dir` 和 `-disable-background-networking`，hosts 一个有效行都没有，自启动里全是 OneDrive、QQ、迅雷这些熟面孔，计划任务里只有 Google 自己的 PlatformExperienceHelper。

然后怀疑扩展。把 `Default\Extensions` 下每个 manifest 都翻了一遍，找 `chrome_url_overrides`、`chrome_settings_overrides`，一个都没有——装的就四个：ChatGPT 侧边栏、一个 C++ 文档搜索、Chrome 商店支付组件，还有一个跨域请求调试的扩展。

到这儿我有点烦了。因为 Chrome 的设置界面里搜 `aitab`，什么都搜不到，`Preferences` 文件里唯一出现 `aitab` 的地方是"你访问过这个网站 8 次"那种统计记录。

## 真正的设置在另一个文件里

Chrome 从很早就把主页、启动页、默认搜索引擎这类偏好挪进了 `Secure Preferences`，并且带 HMAC 校验（`protection.macs` 那一坨）。好处是别的软件不能随便改你的主页，坏处是一旦有工具会重算 HMAC，GUI 里就永远看不出来。

顺手说一句，直接 `Get-Content | ConvertFrom-Json` 这个文件经常报错，不是文件坏了，是 Chrome 正在写它。稳妥的做法是先复制到临时目录再解析。

打开这份 `Secure Preferences`，问题一眼就看见了：

```json
"default_search_provider_data": {
  "template_url_data": {
    "short_name": "百度",
    "keyword": "baidu.com",
    "url": "https://bds.tmdh725.com/#ie={inputEncoding}&wd={searchTerms}",
    "new_tab_url": "https://bds.tmdh725.com/"
  }
}
```

一个叫"百度"、图标也盗用百度的搜索引擎，实际域名是 `bds.tmdh725.com`。关键是那个 `new_tab_url`——Chrome 允许搜索引擎提供"自带的新标签页"，填了它，新标签页就归这个域名管。这跟我一开始以为的"主页被改"完全是两回事，所以设置界面里怎么找都找不到。

我直接请求了一下这个域名，200，页面标题就叫 `AItab`——它自己就是一层壳。历史里 aitab 那条访问的 transition 是"服务器重定向"、`from_visit` 指向前一条，说明浏览器是被 302 过去的。而那个 URL 里带着两个参数：

- `djsource=nO6TdR` — 渠道号
- `search=3037095` — 推广位 ID，正好就是那个假"百度"引擎的编号

说白了就是从这家跳那家，最后落到 360 的推广页上，中间每一跳都有分成。

## 用历史库把每一次触发都对上

光看配置还不够，我想确认"打开就跳"这件事到底是怎么发生的。Chrome 的 History 是个 SQLite，`visits` 表里有个 `transition` 字段，存的是导航类型：

```sql
select v.visit_time, v.transition, u.url
from visits v join urls u on u.id = v.url
order by v.visit_time desc limit 20;
```

`bds.tmdh725.com` 那条是 `0x10000002`，意思是"自动打开、链的起点"；`aitab.360.cn` 那条是 `0xA0000002`，意思是"服务器重定向、链的终点"，`from_visit` 指向前一条。时间戳和每次启动 Chrome 的时刻一一对应，最近一次是 13:08:56。

到这里机制就完整了：Chrome 启动 → 按"新标签页"打开 → 新标签页其实是搜索引擎给的 `new_tab_url` → 落到假百度 → 302 到 aitab。

## 还有个更恶心的：它同步到账号里了

继续在这份 `Secure Preferences` 里翻，发现了 `account_values` 节点，里面存着一份"从云端同步下来的覆盖值"：

```json
"account_values": {
  "homepage": "http://<内网服务>:30033/",
  "session": { "restore_on_startup": 1,
               "startup_urls": ["https://www.google.com/", "about:blank", "http://dhs.cbd263.com/"] }
}
```

这就是为什么我第一轮改完之后，重启一次它又回来了——本地改了，云端还在，Chrome 下次启动照样把覆盖值写回来。顺便说，那串地址是我们公司的一个内网服务，这里给它打个码。改成它，大概是动手的软件顺手抓了当时开着的标签页。

## 动手修

修复的顺序有讲究：**必须先退 Chrome**。Chrome 退出时会把内存里的偏好写回磁盘，开着改等于白改。

然后：

1. 备份 `Preferences` 和 `Secure Preferences`；
2. `Preferences` 里删掉 `default_search_provider` 和 `default_search_provider_data`，Chrome 会自己重建一个默认的（重建出来是 Google）；
3. `Secure Preferences` 里把 `homepage` 清空、`session` 重置成 `restore_on_startup: 5`（新标签页）、删掉 `account_values`，同时把它对应的那几个 MAC 条目也删掉——MAC 留着的话 Chrome 会认为"值被篡改了"再重置一遍，虽然结果一样但会弹通知；
4. `Web Data` 这个 SQLite 里还有一条 keywords 记录（id=1），就是那个假百度，`delete from keywords where id=1`；
5. 注册表里 `HKLM\SOFTWARE\WOW6432Node\Google\Chrome\Extensions\ncennffkjdiamlpmcbajkmaiiiddgioo` 指向 `ChromeExtensionCache\xl_ext_chrome.crx`，删掉，缓存整个目录先移到临时目录而不是直接删。

重启验证：新标签页回 Google 了，历史里再没有 `bds.tmdh725.com`。但 `account_values` 又冒出来了——如前面说的，云端还在。

## 到底是谁干的

这一步最有意思。`Web Data` 的 `keywords` 表虽然不存创建时间（`date_created` 是 0），但 `last_visited` 有：

```
13405357233880352 → 2025-10-19 14:20:33
```

也就是说这个假引擎至少从 2025 年 10 月就在了。

再翻 Chrome 的下载记录，和这台机器的目录创建时间一对：

- 2025-09-02，`360sd`、`360Chrome` 落地
- 2025-09-05 13:22，从 `weishi.360.cn` 下载了 `setupbeta_jisu.exe`（360 极速版卫士），当晚 21:26 出现 `ProgramData\360safe`
- 2025-10-20 13:02，从 `www.360.cn` 下载了 `inst.exe`，当晚 21:03 装出 `360Safe`，21:05 装出 `360se6`（360 安全浏览器）

时间线几乎咬着假引擎的出现时间。360 系的卫士和浏览器都内置"主页锁定 / 默认搜索"这类模块，它们改别的浏览器设置时，走的就是直接改写 `Secure Preferences` 并且重算 HMAC 这条路——不然在 GUI 里早被改回来了。现在 `C:\Program Files (x86)\360` 下面还留着 `360DrvMgr`、`360Safe`、`360sd` 三个空壳目录，服务、进程、卸载项都没了。

要说明的是：Chrome 历史默认只留 90 天，最早只到 2026-08，2025 年当时"哪个进程写的"这个动作已经回放不出来了。所以这是高置信推断，不是现场抓获。

顺便排除一个嫌疑人：迅雷是 2026-06-23 装的，它确实塞了个外部扩展（就是上面删的 `xl_ext_chrome.crx`），但那只是下载支持，跟搜索引擎无关。

## 顺手写了个工具

这种事不值得每次都手工查一遍，所以写了个 PowerShell 脚本，查八个位置：

| 位置 | 为什么查 |
| --- | --- |
| `default_search_provider(_data)` | `new_tab_url` 就是新标签页劫持的入口 |
| `Secure Preferences` 的主页 / 启动页 / `account_values` | 带 HMAC，GUI 里看不见 |
| `Policies\Google\Chrome` | 组策略优先级最高 |
| `Google\Chrome\Extensions` 注册表 | 外部扩展静默安装通道 |
| 快捷方式参数 | `chrome.exe "http://..."`、`-app=`、`--load-extension` |
| 自启动 / 计划任务 / hosts | 常驻改主页、伪造解析 |
| 扩展的 `chrome_url_overrides` | 扩展也能接管新标签页 |
| 安装目录 `master_preferences` | 新 profile 的模板文件 |

默认只读，加 `-Fix` 才动手，动手前备份，退出码 0/1/2 表示干净/可疑/高危。双击 `ScanChromeHijack.cmd` 就能跑。

写的过程中踩了三个坑，值得记一下：

第一个，中文全是乱码。Windows PowerShell 5.1 读 `.ps1` 时，文件没有 BOM 就按 ANSI 解码，UTF-8 的中文直接变方块。解决办法是用 `[IO.File]::WriteAllText($p, $txt, (New-Object Text.UTF8Encoding($true)))` 存成带 BOM 的 UTF-8。这个坑每次都会踩。

第二个更有意思：主页明明没有设置，脚本却报 `homepage = C:\Users\Administrator`。查了半天才发现变量名撞了——`$home` 在 PowerShell 里就是自动变量 `$HOME`，赋值会静默失败，读出来还是用户目录。改叫 `$homeVal` 就好了。

第三个，改 JSON 我一开始用了 `JavaScriptSerializer`，结果在嵌套对象上抛"检测到循环引用"——因为 PowerShell 把嵌套的字典包了一层 `PSObject`。后来换成 `ConvertFrom-Json` / `ConvertTo-Json -Depth 100`，但不敢直接信它，先拿真的 `Secure Preferences` 做了一次往返，用 Python 把两边 `json.load` 出来比对，键、`session`、`account_values`、扩展 MAC 条目数量全都一致，才敢用。

验证是用假数据做的：在 `test-profile` 里造一份带假百度、可疑启动页、`account_values`、假 MAC 的配置，扫描报 3 项高危 1 项注意，`-Fix` 之后文件变成 `{"homepage":"","homepage_is_newtabpage":true,"session":{"restore_on_startup":5}}`，复扫清零。

## 最后一步只能手动

`account_values` 是从 Google 账号同步下来的，删几次都会被拉回去。唯一的断根办法是在 `chrome://settings/syncSetup` 里重置同步。这一步工具做不了，也不该做——删云端数据得自己点。

我把云端那份清掉之后，Chrome 下次启动时自己把本地的 `account_values` 也抹了，`Secure Preferences` 的写入时间停在 13:26。再复查：

- 默认搜索引擎：Google，`new_tab_url` 为空
- 主页、启动页：默认
- 劫持地址最后一次访问：13:08:56，修之前的那一次
- 扫描器：0 高危 0 注意

有点感慨的是，这类劫持现在都不写注册表了。它只改一个你平时不会打开的文件，然后靠云同步活下来——你重装 Chrome 都没用，登录同一个账号，配置自己就回来了。反过来，排查的时候光看注册表和启动项也确实会一无所获，得往 `Secure Preferences` 和 `Web Data` 里钻。

工具留在目录里了，备份文件过两天再删。机器上那三个 360 的空壳目录，等哪天想起来再收拾。
