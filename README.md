# @hualiong/dsh-plugin-suite

一个**安装单元 + 一层补丁**：本包自身不含任何代码，只声明三个原本独立的 DSH 插件为依赖，并把它们各自的行插入 profile 的组合树。

| 行 id | 组件包 | 宿主 | 浏览器 UI |
| --- | --- | --- | --- |
| `mcp-manager` | `@hualiong/dsh-mcp-manager` | MCP 面板的 HTTP/SSE 载体，读写 profile patch 文件 | 设置面板 “MCP” 小节 |
| `chameleon-session-delete` | `@hualiong/dsh-session-delete` | `POST /__chameleon/session/delete` 及其删除链路 | 侧栏会话菜单项 + 确认弹窗 |
| `oauth-mcp-cloudflare`（默认 `disabled: true`） | `@hualiong/dsh-oauth-mcp-client` | OAuth 授权的 Streamable HTTP MCP 客户端桥，**每个服务器一行** | 无 |

三个组件仍然是各自独立的包、各自拥有自己的 `./client` 浏览器产物、各自的 fiber：可单独启停、删除或重排，互不影响。

## 为什么是「聚合包」而不是「单包合并」

DSH 插件详细页的组件行元数据是按**该行声明的模块标识**去读那个模块自己的 `package.json` / `locale/*.json` 的
（`declaredRows()` → `metaOf(row.name)`；客户端渲染 `title = row.meta?.title ?? row.moduleName`）。
所以三行都写同一个包名时，三行必然显示同一个标题和描述，只有行 id 不同（旧版就是这样）。让每行指向**各自的组件包**后：

* 每行显示各自的名称/描述（来自组件包自己的 `locale/<lang>.json`）；
* 每个组件包的 `./client` 各自成为一个独立浏览器模块（旧版是三个 UI 挤在一个 `lib/client.js` 里整包加载）。

## 目录

```
package.json        依赖三个组件包；dsh.bundle.patch = ./cordis.patch.yml；无 icon（用默认图标）
cordis.patch.yml    bundle 层：一个 insert，插入三条行（oauth 行默认 disabled）
locale/en.json      套件卡片的显示名（英文 meta.title / meta.description；锚点文件，必须存在）
locale/zh.json      套件卡片的显示名（中文）
README.md
```

## 安装

profile 的 `package.json`：

```jsonc
{
  "dependencies": {
    "@hualiong/dsh-plugin-suite": "link:C:/Users/64261/Desktop/Workspace/dsh-plugin-suite"
  },
  "dsh": { "profile": { "bundles": [ /* …, */ "@hualiong/dsh-plugin-suite" ] } }
}
```

```powershell
# 三个组件包由本包声明为 dependencies，先装进本包自己的 node_modules
pnpm install --dir C:/Users/64261/Desktop/Workspace/dsh-plugin-suite
dsh plugin --profile desktop install     # 或 pnpm install（profile 目录内）
```

`bundles` 里的顺序即组合顺序；本包自带 `cordis.patch.yml` 会在其中插入下面三条行。

profile 只依赖本包即可，**不要再单独依赖三个组件包**：

- 运行时按「已选中 bundle 的依赖闭包」解析组件（`@deepseek-ai/dsh-app-boot/lib/index.js:819-822` 的 `collectProfileScopePackages`；请求按声明者目录查找，见 `profile-resolution-bootstrap.js:205-211` 的 `packageSearchPaths`），所以本包 `node_modules` 里的三份副本就够用。
- 插件页的「已安装」列表 = `dsh.profile.bundles` ∪ profile 的 `dependencies` ∪ 安装自带依赖（`@deepseek-ai/dsh-plugin-manager/lib/index.js:1459-1469`）。凡是声明了 `dsh.bundle.patch` 的依赖都会占一张卡 —— 在 profile 里再依赖三个组件包，插件页就会多出三张卡（且开关处于「未选中」状态）。
- 改动 profile 的 `dependencies` 后必须**重启 DSH**：运行时解析器拒绝热替换，会报 `profile resolution: removing local package … requires a process restart`（`profile-resolution-bootstrap.js:258`）。
- 想让三个组件的**本地源码改动即时生效**时，把三条 `link:` 加回 profile 的 `dependencies`（插件页会重新出现那三张卡），改完再删掉。

## 行长什么样

```yaml
# 包自带层（cordis.patch.yml）
- insert:
    - id: mcp-manager
      name: '@hualiong/dsh-mcp-manager'
    - id: chameleon-session-delete
      name: '@hualiong/dsh-session-delete'
    # 每个 OAuth 服务器一行；默认禁用，绝不因「装了套件」而触发授权流程
    - id: oauth-mcp-cloudflare
      name: '@hualiong/dsh-oauth-mcp-client'
      disabled: true
      config:
        serverName: Cloudflare
        url: https://mcp.cloudflare.com/mcp
        credentialRef: CLOUDFLARE_MCP_OAUTH
        callbackPort: 45871
        authorizationTimeoutMs: 600000
        failOnStartupError: false
```

profile 自己的 patch 文件按 id 覆盖这些行来补配置 / 打开开关（patch 会替换该行的**整个** `config`，所以要把它拥有的键写全）：

```yaml
- id: mcp-manager
  name: '@hualiong/dsh-mcp-manager'
  config:
    patchFile: C:\Users\64261\.dsh\profiles\desktop\cordis.patch.yml
  disabled: false

- id: oauth-mcp-cloudflare
  name: '@hualiong/dsh-oauth-mcp-client'
  disabled: false          # 本 profile 打开这个 OAuth 服务器
  config: { serverName: Cloudflare, url: …, credentialRef: …, callbackPort: 45871, authorizationTimeoutMs: 600000, failOnStartupError: false }
```

## 显示名称（中英双语）

DSH 读取显示元数据的方式是：先把 `${包名}/locale/en.json` 当作**锚点**解析，再扫描该文件所在目录下所有 `*.json` 作为各语言词典；
显示时 `ctx.locale.resolveText(text)` 以 **`en` 键作为兜底种子**，再沿当前语言的回退链查找。
所以四个包都提供规范的 `locale/en.json`（英文）+ `locale/zh.json`（中文），并按当前界面语言显示：

| 包 | 详细页标题（zh / en） | 图标 |
| --- | --- | --- |
| `@hualiong/dsh-plugin-suite` | DSH 插件套件 / DSH Plugin Suite | 无（默认图标） |
| `@hualiong/dsh-mcp-manager` | MCP 管理器 / MCP Manager | 无（默认图标） |
| `@hualiong/dsh-oauth-mcp-client` | OAuth MCP 客户端 / OAuth MCP Client | 无（默认图标） |
| `@hualiong/dsh-session-delete` | 会话删除 / Session Delete | 无（默认图标） |

* 改标题/描述：编辑对应包的 `locale/en.json`（英文）与 `locale/zh.json`（中文）的 `meta.title` / `meta.description`；
  新增语言 = 在同目录再加一份 `<lang>.json`（文件名必须是合法语言 id），`en` 必须始终存在；
* 这些文件必须能被解析到 —— 三个组件包导出 `"./package.json"`、`"./locale/en.json"`、`"./locale/zh.json"`；
  套件包**有意不声明 `exports`**，因此 `${包名}/package.json`、`${包名}/locale/*.json` 与 `${包名}/cordis.patch.yml` 都能按路径解析；
* 四个包都**不声明 `icon`**，所以卡片与每个组件行都用默认图标。

## 单独启停 / 卸载

* **停用某个组件**：在 profile patch 里给该行加/改最后一条匹配覆盖项的 `disabled: true`，或直接删掉该行。
* **停用整包**：把 `@hualiong/dsh-plugin-suite` 从 `dsh.profile.bundles` 移除（依赖可保留，官方推荐的“关闭保留依赖”语义）。
  本机 desktop profile 就是这么停用原来那三个插件的：三个旧包仍在 `dependencies`，但不在 `bundles` 里。
* **卸载**：先移出 `bundles`，再 `pnpm remove`。

## 发布（npm）

四个包的 registry 现状（查询于 2026-10-11）：

| 包 | registry | 发布方式 |
| --- | --- | --- |
| `@hualiong/dsh-mcp-manager` | **已发布 `0.1.6`**（`0.1.5` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38121187747` 成功 |
| `@hualiong/dsh-oauth-mcp-client` | **已发布 `0.1.1`**（`0.1.0` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38121182396` 成功 |
| `@hualiong/dsh-session-delete` | **已发布 `0.3.5`**（`0.3.3`、`0.3.4` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38118548308` 成功 |
| `@hualiong/dsh-plugin-suite` | **已发布 `1.1.0`** | 首发 CI run `38121812115` 成功（当时用一次性 `NPM_TOKEN` + `id-token: write` 签出 provenance）；workflow 随后已切成 OIDC 并删掉 secret |

四个包的 workflow 现在**都是 OIDC 路线**（无任何 secret，四个仓库的 `gh secret list` 均为空）：

```yaml
on:
  push:
    tags: ['v*']
permissions:
  contents: read
  id-token: write
# setup-node 故意不写 registry-url（它会往 .npmrc 写空的 _authToken 占位符，遮蔽 OIDC 凭据）
npm install -g npm@latest     # trusted publishing 需要 npm >= 11.5.1
npm publish --access public --provenance
```

发布姿势：**版本号写进 `package.json` → 打同名 tag → 推 tag 触发 CI → 用 gh 观察**（**不建 GitHub Release**）：

```powershell
gh auth setup-git                                    # 让 git push 用 gh 的凭据

git -C dsh-mcp-manager tag v0.1.6                    # tag 必须等于 package.json 的 version，CI 会校验
git -C dsh-mcp-manager push origin fix/dsh-0.2.0-rc.2 v0.1.6

git -C dsh-oauth-mcp-client tag v0.1.1
git -C dsh-oauth-mcp-client push origin main v0.1.1

git -C dsh-plugin-suite tag v1.1.0
git -C dsh-plugin-suite push origin main v1.1.0

gh run watch --repo hualiong/dsh-mcp-manager $(gh run list --repo hualiong/dsh-mcp-manager --limit 1 --json databaseId -q '.[0].databaseId')
```

每个包在 npm 网页上各有一次性的 trusted publisher 绑定（Settings → Trusted Publisher → GitHub Actions）：
`organization/user: hualiong`、`repository: <仓库名>`、`workflow filename: publish.yml`、`Environment name` **留空**、勾 **Allow npm publish**。
命令行等价物 `npm trust github <包名> --file publish.yml --repo hualiong/<repo> --allow-publish` 也可，
但它属于账号变更操作、**必须交互式 2FA**（bypass-2FA token 会被 npm 拒绝），只能由账号主人执行一次。
绑定好之后**不要再配 `NPM_TOKEN` secret**，也不要把 bypass-2FA token 交给 CI。
（`dsh-mcp-manager`、`dsh-oauth-mcp-client`、`dsh-session-delete` 已绑定；`dsh-plugin-suite` 的 `1.1.0` 是先用一次性
token 发布的，**绑定它的 trusted publisher 之后，下一次推 tag 才会走 OIDC**。）

`0.1.6` / `0.1.1` 顺带修掉的小瑕疵：

* `dsh-mcp-manager`：删掉指向从未构建的 `lib/types/` 的 `types` 与 `exports[].types`；
  补 `keywords`；删掉 `pnpm.onlyBuiltDependencies`（pnpm 11 已不读 `package.json` 的 `pnpm` 字段，改用
  `pnpm-workspace.yaml` 的 `allowBuilds`）；删掉项目 `.npmrc`（它的 pnpm-only 键让 npm 打
  `npm warn Unknown project config "autoInstallPeers"`，而 npm 12 会直接报错）。
* `dsh-oauth-mcp-client`：补上缺的 `author`。
* 两个 `publish.yml` 都加了 `pnpm test`（oauth 另加 `pnpm typecheck`）作为发布前置门槛。

首次发布的包在 registry 上会留下一个 `0.0.0-stage` 版本，那是 npm 处理首包时写入的占位版本，
`dist-tags.latest` 指向真实版本，不影响安装。

发布内容已用 `npm pack --dry-run --json --ignore-scripts` 逐包核对：
mcp-manager 10 项（`lib/index.js`、`lib/client.js` + 两个 sourcemap、`cordis.patch.yml`、`locale/{en,zh}.json`、LICENSE/README/package.json）、
oauth 25 项（`lib/**` 的 js/d.ts/map、`dsh-oauth-mcp.cordis.yml`、`locale/{en,zh}.json`、README）、
session-delete 7 项（`src/{index,client}.js`、`cordis.patch.yml`、`locale/{en,zh}.json`，已无 `icon.svg`）、
suite 6 项（`cordis.patch.yml`、`package.json`、`README.md`、`LICENSE`、`locale/{en,zh}.json`）。

一手小坑：**带 token 的 workflow 里 `npm publish --provenance` 同样需要 `id-token: write`**，否则 CI 会以
`npm error Provenance generation in GitHub Actions requires "write" access to the "id-token" permission` 直接失败。

发布后别人的 profile 只需

```jsonc
{
  "dependencies": { "@hualiong/dsh-plugin-suite": "1.1.0" },
  "dsh": { "profile": { "bundles": [ /* …, */ "@hualiong/dsh-plugin-suite" ] } }
}
```

即可 —— 三个组件包会被套件的 `dependencies` 自动拉进来，不必逐个安装。

## 从原来的三个插件迁移

| 原行 | 新行 |
| --- | --- |
| `id: mcp-manager`, `name: '@hualiong/dsh-mcp-manager'` | 同名 id、同一个 `name`（现在由套件插入），profile 覆盖行只提供 `config.patchFile` |
| `id: chameleon-session-delete`, `name: '@hualiong/dsh-session-delete'` | 同名 id、同一个 `name`（现在由套件插入，原先由该包自己的 patch 插入） |
| `id: oauth-mcp-cloudflare`, `name: '@hualiong/dsh-oauth-mcp-client'` | 同名 id、同一个 `name`（原先写在 profile 的 `insert:` 里，现在由套件插入并在本 profile 覆盖为 `disabled: false`） |

## 验证记录（本机 desktop profile）

* 元数据：四个包的 `package.json` + `locale/en.json` + `locale/zh.json` 均可按 profile 路径解析，
  词典为 `en+zh`、标题与描述在两种语言下都成立（脚本模拟 `readPluginMeta` 的锚点/扫描/兜底逻辑，全部通过）；
  四个包的 `icon` 均为空（用默认图标）；套件 `v1.1.0` 的 `dsh.client` 不存在（纯补丁层，合法；官方 `@deepseek-ai/dsh-web-app` 同理）、
  `dsh.bundle.patch = ./cordis.patch.yml`。
* 组合层：一次性探针 profile（`bundles = dsh-base + dsh-web-app + 本包`）`--dump-config` 退出码 0，逐行确认三条行
  （`mcp-manager`、`chameleon-session-delete`、`oauth-mcp-cloudflare`）均来自本包，且 `oauth-mcp-cloudflare` 为 `disabled`。
  Electron 独占的 desktop profile 本身不允许 `--dump-config`。
* 真机启动（探针 profile，`dsh <profile> --no-open --port 51410`）：**无 “did not activate” 警告**；
  * 浏览器清单（`__DSH_BOOT__`，67 条）里 `@hualiong/dsh-mcp-manager` 与 `@hualiong/dsh-session-delete` 各自是一个模块，
    且**不含 `@hualiong/dsh-plugin-suite`** ⇒ 两个 UI 各自独立加载；
  * `POST /mcp-manager/list`（body `{"payload":{}}`）→ `200 {"ok":true,…}`，列出 `id: oauth-mcp-cloudflare`、
    `ownerPackage: '@hualiong/dsh-oauth-mcp-client'`、`enabled: false`（= 套件层的 `disabled: true` 生效）；
  * `POST /__chameleon/session/delete` → `400 {"error":"invalid session id: …"}` ⇒ 组件路由已挂载，无重复注册。
* 迁移期间发现并修复：`@hualiong/dsh-session-delete` 的源码 `import { dshHomePath } from '@deepseek-ai/dsh-home-paths'`
  从未在 `package.json` 里声明（该包原本零依赖），一旦从 npm 安装改为 `link:` 指向工作区目录就会
  `ERR_MODULE_NOT_FOUND`。已在 `dsh-session-delete/package.json` 的 `devDependencies` 里补上
  `"@deepseek-ai/dsh-home-paths": "0.2.0-rc.2"` 并 `pnpm install`（与 `dsh-mcp-manager` 的既有约定一致）。

## 已知限制

* OAuth 桥是「每个服务器一行」，套件里只提供一条**默认禁用**的示例行；启用它需要在 profile patch 里覆盖，或由 MCP 面板写入。
* 三个组件的 UI 不再共享一个浏览器产物，但 `dsh.client` 的排序只作用于单个包内部：跨组件的加载顺序由各自的 `inject` 决定。
* 组件内部的日志前缀沿用各组件包自己的前缀（`dsh-mcp-manager(...)` 等）。
* `@hualiong/dsh-mcp-manager` 的 `lib/` 里只有 `index.js` / `client.js` 与两个 sourcemap，没有 `lib/types/`，
  因此它**不再对外声明类型**（`0.1.6` 起删掉了指向不存在文件的 `types` / `exports[].types`，而不是留一个坏指针）。
  运行时不受影响（DSH Loader 只按 ESM import `lib/index.js` / `lib/client.js`）；要让 TypeScript 使用者拿到声明，
  得先让 `scripts/build.mjs` 产出 `.d.ts`（其 `tsconfig.json` 的 `paths` 指向 DSH checkout，`tsc --emitDeclarationOnly` 目前无法直接跑通）。
