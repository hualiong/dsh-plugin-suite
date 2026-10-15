# @hualiong/dsh-plugin-suite

一个**安装单元 + 一层补丁**：本包自身不含任何代码，只声明四个原本独立的 DSH 插件为依赖，并把其中三个的行插入 profile 的组合树（OAuth 桥只装、不带行 —— 服务器行由你添加，见下）。

| 行 id | 组件包 | 宿主 | 浏览器 UI |
| --- | --- | --- | --- |
| `mcp-manager` | `@hualiong/dsh-mcp-manager` | MCP 面板的 HTTP/SSE 载体，读写 profile patch 文件 | 设置面板 “MCP” 小节 |
| `chameleon-session-delete` | `@hualiong/dsh-session-delete` | `POST /__chameleon/session/delete` 及其删除链路 | 侧栏会话菜单项 + 确认弹窗 |
| （套件**不预置**服务器行） | `@hualiong/dsh-oauth-mcp-client` | OAuth 授权的 Streamable HTTP MCP 客户端桥，**每个服务器一行**；装好了，但它自己不会启动任何服务器 | 无 |
| `cost-lite`（默认 `disabled: true`） | `@hualiong/dsh-cost-lite` | `GET /__cost-lite/usage?sessionId=…` 的计费快照 | 输入框停靠栏的会话费用徽标 + 助手消息操作行的单轮徽标 |

四个组件仍然是各自独立的包、各自拥有自己的 `./client` 浏览器产物、各自的 fiber：可单独启停、删除或重排，互不影响。

## 为什么是「聚合包」而不是「单包合并」

DSH 插件详细页的组件行元数据是按**该行声明的模块标识**去读那个模块自己的 `package.json` / `locale/*.json` 的
（`declaredRows()` → `metaOf(row.name)`；客户端渲染 `title = row.meta?.title ?? row.moduleName`）。
所以多行都写同一个包名时，各行必然显示同一个标题和描述，只有行 id 不同（旧版就是这样）。让每行指向**各自的组件包**后：

* 每行显示各自的名称/描述（来自组件包自己的 `locale/<lang>.json`）；
* 每个组件包的 `./client` 各自成为一个独立浏览器模块（旧版是几个 UI 挤在一个 `lib/client.js` 里整包加载）。

## 目录

```
package.json        依赖四个组件包；dsh.bundle.patch = ./cordis.patch.yml；无 icon（用默认图标）
cordis.patch.yml    bundle 层：一个 insert，插入三行（mcp-manager / session-delete / cost-lite；cost-lite 默认 disabled）
locale/en.json      套件卡片的显示名（英文 meta.title / meta.description；锚点文件，必须存在）
locale/zh.json      套件卡片的显示名（中文）
README.md
```

## 安装

从 registry 安装（推荐）——一条命令同时装依赖并登记 bundle 层：

```powershell
dsh plugin --profile desktop add @hualiong/dsh-plugin-suite
```

`dsh plugin --profile <name> <pnpm 参数…>` 把参数转发给 profile 目录里的 pnpm；装完后
`@deepseek-ai/dsh-plugin-manager` 的 `reconcile()` 会读新依赖的 `dsh.bundle`，把包追加进 `dsh.profile.bundles`
（包没声明 `dsh.bundle` 时只输出一条 warning，不登记）。

手工等价做法（profile 的 `package.json`）：

```jsonc
{
  "dependencies": {
    "@hualiong/dsh-plugin-suite": "^1.3.1"
  },
  "dsh": { "profile": { "bundles": [ /* …, */ "@hualiong/dsh-plugin-suite" ] } }
}
```

本地开发（把工作区目录链进 profile，改源码即时生效）：

```jsonc
{
  "dependencies": {
    "@hualiong/dsh-plugin-suite": "link:C:/Users/64261/Desktop/Workspace/dsh-plugin-suite"
  },
  "dsh": { "profile": { "bundles": [ /* …, */ "@hualiong/dsh-plugin-suite" ] } }
}
```

```powershell
# 四个组件包由本包声明为 dependencies，先装进本包自己的 node_modules
pnpm install --dir C:/Users/64261/Desktop/Workspace/dsh-plugin-suite
dsh plugin --profile desktop install     # 或 pnpm install（profile 目录内）
```

`bundles` 里的顺序即组合顺序；本包自带 `cordis.patch.yml` 会在其中插入下面三行。
**本包不预置任何 MCP 服务器行** —— OAuth 桥装好了，但服务器行要你自己加（MCP 面板，或 profile 的 `insert:`）。

profile 只依赖本包即可，**不要再单独依赖四个组件包**：

- 运行时按「已选中 bundle 的依赖闭包」解析组件（`@deepseek-ai/dsh-app-boot/lib/index.js:819-822` 的 `collectProfileScopePackages`；请求按声明者目录查找，见 `profile-resolution-bootstrap.js:205-211` 的 `packageSearchPaths`），所以本包 `node_modules` 里的四份副本就够用。
- 插件页的「已安装」列表 = `dsh.profile.bundles` ∪ profile 的 `dependencies` ∪ 安装自带依赖（`@deepseek-ai/dsh-plugin-manager/lib/index.js:1459-1469`）。凡是声明了 `dsh.bundle.patch` 的依赖都会占一张卡 —— 在 profile 里再依赖四个组件包，插件页就会多出四张卡（且开关处于「未选中」状态）。
- 改动 profile 的 `dependencies` 后必须**重启 DSH**：运行时解析器拒绝热替换，会报 `profile resolution: removing local package … requires a process restart`（`profile-resolution-bootstrap.js:258`）。
- 想让四个组件的**本地源码改动即时生效**时，把四条 `link:` 加回 profile 的 `dependencies`（插件页会重新出现那四张卡），改完再删掉。

## 行长什么样

```yaml
# 包自带层（cordis.patch.yml）
- insert:
    - id: mcp-manager
      name: '@hualiong/dsh-mcp-manager'
    - id: chameleon-session-delete
      name: '@hualiong/dsh-session-delete'
    # 默认禁用：计费是 opt-in，装套件不改变会话显示
    - id: cost-lite
      name: '@hualiong/dsh-cost-lite'
      disabled: true
# 注意：这里没有 MCP 服务器行。OAuth 桥（@hualiong/dsh-oauth-mcp-client）是依赖、已经装好，
# 但“行”才是启动服务器的东西 —— 预置一条就等于给每个装套件的 profile 塞一台没人要的服务器。
# 要用时在设置 → MCP 面板点“新增”，或自己插一条（形状见下）。
```

profile 自己的 patch 文件按 id 覆盖这些行来补配置 / 打开开关（patch 会替换该行的**整个** `config`，所以要把它拥有的键写全）。
profile patch 是最后一层，它写的 `disabled` 最终生效 —— 例如套件把 `cost-lite` 默认关掉，而这个 profile 要打开它：

```yaml
- id: mcp-manager
  name: '@hualiong/dsh-mcp-manager'
  config:
    patchFile: C:\Users\64261\.dsh\profiles\desktop\cordis.patch.yml
  disabled: false

- id: cost-lite
  name: '@hualiong/dsh-cost-lite'
  disabled: false          # 这个 profile 打开计费

# 套件不预置 MCP 服务器行，所以服务器行要自己插（MCP 面板的“新增”替你写的就是这个形状）
- insert:
    - id: oauth-mcp-cloudflare
      name: '@hualiong/dsh-oauth-mcp-client'
      config:
        serverName: Cloudflare
        url: https://mcp.cloudflare.com/mcp
        credentialRef: CLOUDFLARE_MCP_OAUTH
        callbackPort: 45871
        authorizationTimeoutMs: 600000
        failOnStartupError: false
```

## 显示名称（中英双语）

DSH 读取显示元数据的方式是：先把 `${包名}/locale/en.json` 当作**锚点**解析，再扫描该文件所在目录下所有 `*.json` 作为各语言词典；
显示时 `ctx.locale.resolveText(text)` 以 **`en` 键作为兜底种子**，再沿当前语言的回退链查找。
所以五个包都提供规范的 `locale/en.json`（英文）+ `locale/zh.json`（中文），并按当前界面语言显示：

| 包 | 详细页标题（zh / en） | 图标 |
| --- | --- | --- |
| `@hualiong/dsh-plugin-suite` | DSH 插件套件 / DSH Plugin Suite | 无（默认图标） |
| `@hualiong/dsh-mcp-manager` | MCP 管理器 / MCP Manager | 无（默认图标） |
| `@hualiong/dsh-oauth-mcp-client` | OAuth MCP 客户端 / OAuth MCP Client | 无（默认图标） |
| `@hualiong/dsh-session-delete` | 会话删除 / Session Delete | 无（默认图标） |
| `@hualiong/dsh-cost-lite` | 费用计量 / Cost Meter | 无（默认图标） |

* 改标题/描述：编辑对应包的 `locale/en.json`（英文）与 `locale/zh.json`（中文）的 `meta.title` / `meta.description`；
  新增语言 = 在同目录再加一份 `<lang>.json`（文件名必须是合法语言 id），`en` 必须始终存在；
* 这些文件必须能被解析到 —— 四个组件包导出 `"./package.json"`、`"./locale/en.json"`、`"./locale/zh.json"`；
  套件包**有意不声明 `exports`**，因此 `${包名}/package.json`、`${包名}/locale/*.json` 与 `${包名}/cordis.patch.yml` 都能按路径解析；
* 五个包都**不声明 `icon`**，所以卡片与每个组件行都用默认图标。

## 单独启停 / 卸载

* **停用某个组件**：在 profile patch 里给该行加/改最后一条匹配覆盖项的 `disabled: true`，或直接删掉该行
  （`cost-lite` 默认 `disabled: true`，其它两行默认启用；套件不预置 MCP 服务器行，服务器行完全由你添加/删除）。
* **停用整包**：把 `@hualiong/dsh-plugin-suite` 从 `dsh.profile.bundles` 移除（依赖可保留，官方推荐的“关闭保留依赖”语义）。
  本机 desktop profile 就是这么停用原来那三个插件的：旧包仍在 `dependencies`，但不在 `bundles` 里。
* **卸载**：先移出 `bundles`，再 `pnpm remove`。

## 发布（npm）

五个包的 registry 现状（查询于 2026-10-14）：

| 包 | registry | 发布方式 |
| --- | --- | --- |
| `@hualiong/dsh-mcp-manager` | **已发布 `0.1.7`**（`0.1.5`、`0.1.6` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC） |
| `@hualiong/dsh-oauth-mcp-client` | **已发布 `0.1.1`**（`0.1.0` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38121182396` 成功 |
| `@hualiong/dsh-session-delete` | **已发布 `0.3.6`**（`0.3.3`、`0.3.4`、`0.3.5` 为历史版本） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38123210607` 成功 |
| `@hualiong/dsh-cost-lite` | **已发布 `0.1.2`**（`0.1.1` 为历史版本；独立仓库 `hualiong/dsh-cost-lite`） | `publish.yml` 的 npm trusted publishing（OIDC）；CI run `38591985731` 成功（带 provenance，`logIndex=3251629629`） |
| `@hualiong/dsh-plugin-suite` | **已发布 `1.3.1`**（`1.1.0`、`1.1.1`、`1.2.1`、`1.3.0` 为历史版本；`1.2.0` 曾误发并撤回） | `publish.yml` 的 npm trusted publishing（OIDC） |

五个包的 workflow 现在**都是 OIDC 路线**（无任何 secret，各仓库的 `gh secret list` 均为空）：

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

套件自己的 `publish.yml` 在发布前还会校验内容：`cordis.patch.yml` / `locale/{en,zh}.json` / `README.md` / `LICENSE` 必须存在，
且 `cordis.patch.yml` 里 `name: '@hualiong/` 的出现次数不得少于 3（当前正好 3 行；注释里的示例故意写成占位符，免得把这条检查喂饱）。

发布姿势：**版本号写进 `package.json` → 打同名 tag → 推 tag 触发 CI → 用 gh 观察**（**不建 GitHub Release**）：

```powershell
gh auth setup-git                                    # 让 git push 用 gh 的凭据

git -C dsh-cost-lite tag v0.1.2                      # tag 必须等于 package.json 的 version，CI 会校验
git -C dsh-cost-lite push origin main v0.1.2

git -C dsh-plugin-suite tag v1.3.1
git -C dsh-plugin-suite push origin main v1.3.1

gh run watch --repo hualiong/dsh-plugin-suite $(gh run list --repo hualiong/dsh-plugin-suite --limit 1 --json databaseId -q '.[0].databaseId')
```

每个包在 npm 网页上各有一次性的 trusted publisher 绑定（Settings → Trusted Publisher → GitHub Actions）：
`organization/user: hualiong`、`repository: <仓库名>`、`workflow filename: publish.yml`、`Environment name` **留空**、勾 **Allow npm publish**。
命令行等价物 `npm trust github <包名> --file publish.yml --repo hualiong/<repo> --allow-publish` 也可，
但它属于账号变更操作、**必须交互式 2FA**（bypass-2FA token 会被 npm 拒绝），只能由账号主人执行一次。
绑定好之后**不要再配 `NPM_TOKEN` secret**，也不要把 bypass-2FA token 交给 CI。

`1.3.1` 的内容变化：

* **删掉套件预置的 MCP 服务器行**。`1.3.0` 把 `oauth-mcp-cloudflare` 改成默认启用后，任何人装套件都会凭空得到一台
  指向 `https://mcp.cloudflare.com/mcp` 的 Cloudflare MCP（MCP 面板按组合树里的行列服务器，所以它立刻出现在「设置 → MCP」里）。
  这不是 `@hualiong/dsh-mcp-manager` 的行为 —— 它的 patch 只插自己一行，README 里的 Cloudflare 只是 `credentialRef` 推导示例；
  行是套件插的。`1.3.1` 把这一行彻底删掉：OAuth 桥仍作为依赖装好（要用随时能用），但**不再有任何预置服务器**。

`1.3.0` 的内容变化：

* 新增组件行 `cost-lite`（`@hualiong/dsh-cost-lite@^0.1.2`），**默认 `disabled: true`**：安装套件不改变会话显示，
  要计费徽标得在插件页打开或由 profile patch 覆盖成 `disabled: false`；
* OAuth 示例行 `oauth-mcp-cloudflare` 当时被改成默认启用（去掉 `disabled: true`），**`1.3.1` 已把这一行删除**（原因见上）；
* 描述与中文/英文卡片文案同步为「四个组件」。

`0.1.2`（cost-lite 独立仓库）的内容变化：计费口径收紧为**只按精确 `(provider, model)` 计价**，删掉整条 provider 级兜底链
（`PROVIDER_FALLBACK` / `DEFAULT_FALLBACK_PROVIDERS` / `config.allowFallback` / `config.fallbackProviders`），
价目表只剩 4 行；官方已下架的 `deepseek-v4-flash` / `deepseek-v4-flash-vision-exp` 不再登记。

`0.3.6` / `1.1.1` 修掉的问题：

* `dsh-session-delete` 的 host half 在 `src/index.js` 里 `import '@deepseek-ai/dsh-home-paths'`，
  但该包此前只把它写在 `devDependencies`。源码 checkout（`link:` 安装）能跑，**从 registry 安装时这个依赖不会被装**，
  Cordis 于是报 `1 entry did not activate chameleon-session-delete (@hualiong/dsh-session-delete): failed to import`
  （`ERR_MODULE_NOT_FOUND`）。`0.3.6` 把它移进 `dependencies`（`0.2.0-rc.2`），套件的依赖下限同步提到 `^0.3.6`。
  这个坑只在组件作为套件依赖从 registry 装时才暴露：`link:` 到工作区目录时，工作区自己的 `node_modules` 恰好能解析它。
  `dsh-mcp-manager` / `dsh-oauth-mcp-client` / `dsh-cost-lite` 不 import 该包，所以一直正常（它们的客户端 half 用的
  `@deepseek-ai/dsh-client-*`、`@deepseek-ai/cordis`、`react` 由 DSH 运行时的解析表提供）。

`0.1.6` / `0.1.1` 顺带修掉的小瑕疵：

* `dsh-mcp-manager`：删掉指向从未构建的 `lib/types/` 的 `types` 与 `exports[].types`；
  补 `keywords`；删掉 `pnpm.onlyBuiltDependencies`（pnpm 11 已不读 `package.json` 的 `pnpm` 字段，改用
  `pnpm-workspace.yaml` 的 `allowBuilds`）；删掉项目 `.npmrc`（它的 pnpm-only 键让 npm 打
  `npm warn Unknown project config "autoInstallPeers"`，而 npm 12 会直接报错）。
* `dsh-oauth-mcp-client`：补上缺的 `author`。
* 两个 `publish.yml` 都加了 `pnpm test`（oauth 另加 `pnpm typecheck`）作为发布前置门槛。

首次发布的包在 registry 上会留下一个 `0.0.0-stage` 版本，那是 npm 处理首包时写入的占位版本，
`dist-tags.latest` 指向真实版本，不影响安装（`dsh-cost-lite` 的 `0.0.0-stage` 仍在 versions 列表里）。

发布内容已用 `npm pack --dry-run --json --ignore-scripts` 逐包核对：
mcp-manager 10 项（`lib/index.js`、`lib/client.js` + 两个 sourcemap、`cordis.patch.yml`、`locale/{en,zh}.json`、LICENSE/README/package.json）、
oauth 25 项（`lib/**` 的 js/d.ts/map、`dsh-oauth-mcp.cordis.yml`、`locale/{en,zh}.json`、README）、
session-delete 7 项（`src/{index,client}.js`、`cordis.patch.yml`、`locale/{en,zh}.json`，已无 `icon.svg`）、
cost-lite 9 项（`src/{index,client,pricing}.js`、`cordis.patch.yml`、`locale/{en,zh}.json`、LICENSE/README/package.json）、
suite 6 项（`cordis.patch.yml`、`package.json`、`README.md`、`LICENSE`、`locale/{en,zh}.json`）。

一手小坑：**带 token 的 workflow 里 `npm publish --provenance` 同样需要 `id-token: write`**，否则 CI 会以
`npm error Provenance generation in GitHub Actions requires "write" access to the "id-token" permission` 直接失败。
另一个坑：`npm view <包>@<新版本>` 在发布后几分钟内可能仍报 `E404 No match found for version …`（CDN/元数据缓存），
别据此判定发布失败 —— 看 CI 日志，或带 cache-bust 查 `https://registry.npmjs.org/<编码后的包名>?t=<随机数>`。

发布后别人的 profile 只需

```jsonc
{
  "dependencies": { "@hualiong/dsh-plugin-suite": "1.3.1" },
  "dsh": { "profile": { "bundles": [ /* …, */ "@hualiong/dsh-plugin-suite" ] } }
}
```

即可 —— 四个组件包会被套件的 `dependencies` 自动拉进来，不必逐个安装。

## 从原来的独立插件迁移

| 原行 | 新行 |
| --- | --- |
| `id: mcp-manager`, `name: '@hualiong/dsh-mcp-manager'` | 同名 id、同一个 `name`（现在由套件插入），profile 覆盖行只提供 `config.patchFile` |
| `id: chameleon-session-delete`, `name: '@hualiong/dsh-session-delete'` | 同名 id、同一个 `name`（现在由套件插入，原先由该包自己的 patch 插入） |
| `id: oauth-mcp-cloudflare`, `name: '@hualiong/dsh-oauth-mcp-client'` | 同名 id、同一个 `name`；**这一行不再由套件插入**（`1.3.1` 起）。套件只把它作为依赖装好，行本身仍由 profile 的 `insert:` 或 MCP 面板维护 —— 原来怎么写就怎么留 |
| `id: cost-lite`, `name: '@hualiong/dsh-cost-lite'` | 同名 id、同一个 `name`（原先由该包自己的 patch 插入；套件里默认 `disabled: true`，profile 覆盖行按需打开） |

## 验证记录

* **`1.3.1` 组合层（本机，2026-10-14）**：一次性探针 profile（`dsh suite-verify --from-default-profile web --dump-config` 创建，
  `bundles = dsh-base + dsh-web-app`），`dsh plugin --profile suite-verify add "link:…/dsh-plugin-suite"` 退出码 0，
  随后 `dsh --profile suite-verify --dump-config` 退出码 0（1267 行），逐行确认本包插入的行**只有三行**：
  `mcp-manager`、`chameleon-session-delete`、`cost-lite` + `disabled: true`（默认关闭）；
  整份组合树里 **`oauth` 出现 0 次** ⇒ 不再预置任何 MCP 服务器行（服务器行由用户自己加）。探针 profile 已整目录删除。
  `cordis.patch.yml` 里 `name: '@hualiong/` 出现次数 = 3（正好等于真实行数，注释里的示例写成占位符，不参与计数）。
* **`1.3.0` 组合层（本机，2026-10-14）**：一次性探针 profile（`dsh suite-verify --from-default-profile web --dump-config` 创建，
  `bundles = dsh-base + dsh-web-app`），`dsh plugin --profile suite-verify add "link:…/dsh-plugin-suite"` 退出码 0，
  随后 `dsh --profile suite-verify --dump-config` 退出码 0，逐行确认本包插入的四行：
  `mcp-manager`、`chameleon-session-delete`、`oauth-mcp-cloudflare`（**无 `disabled` 字段 ⇒ 默认启用**）、
  `cost-lite` + `disabled: true`（**默认关闭**）。探针 profile 已整目录删除。
  本轮**没有**做真机 boot 冒烟：`pnpm install`（非 `--lockfile-only`）在 suite 目录被供货链策略拦下
  （见下面的已知坑），四个组件包没进 suite 的 `node_modules`。
* **`1.1.0`/`1.1.1` 元数据**：五个包的 `package.json` + `locale/en.json` + `locale/zh.json` 均可按 profile 路径解析，
  词典为 `en+zh`、标题与描述在两种语言下都成立（脚本模拟 `readPluginMeta` 的锚点/扫描/兜底逻辑，全部通过）；
  五个包的 `icon` 均为空（用默认图标）；套件的 `dsh.client` 不存在（纯补丁层，合法；官方 `@deepseek-ai/dsh-web-app` 同理）、
  `dsh.bundle.patch = ./cordis.patch.yml`。
* **`1.1.0` 真机启动（探针 profile，`dsh <profile> --no-open --port 51410`）**：**无 “did not activate” 警告**；
  * 浏览器清单（`__DSH_BOOT__`，67 条）里 `@hualiong/dsh-mcp-manager` 与 `@hualiong/dsh-session-delete` 各自是一个模块，
    且**不含 `@hualiong/dsh-plugin-suite`** ⇒ 两个 UI 各自独立加载；
  * `POST /mcp-manager/list`（body `{"payload":{}}`）→ `200 {"ok":true,…}`，列出 `id: oauth-mcp-cloudflare`、
    `ownerPackage: '@hualiong/dsh-oauth-mcp-client'`、`enabled: false`（当时套件层还是 `disabled: true`）；
  * `POST /__chameleon/session/delete` → `400 {"error":"invalid session id: …"}` ⇒ 组件路由已挂载，无重复注册。
* 迁移期间发现并修复：`@hualiong/dsh-session-delete` 的源码 `import { dshHomePath } from '@deepseek-ai/dsh-home-paths'`
  从未在 `package.json` 里声明（该包原本零依赖），一旦从 npm 安装改为 `link:` 指向工作区目录就会
  `ERR_MODULE_NOT_FOUND`。已在 `dsh-session-delete/package.json` 的 `devDependencies` 里补上
  `"@deepseek-ai/dsh-home-paths": "0.2.0-rc.2"` 并 `pnpm install`（与 `dsh-mcp-manager` 的既有约定一致）。

## 已知限制

* **本地 `pnpm install` 会被供货链策略拦下**：在 suite 目录跑真装时遇到过
  `[ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION] 1 lockfile entries failed verification: @hualiong/dsh-mcp-manager@0.1.7 was published at
  2026-10-13T16:11:52.000Z, within the minimumReleaseAge cutoff (2026-10-13T15:49:04.687Z)` ⇒ 退出码 1、`node_modules` 不生成。
  `pnpm-workspace.yaml` 里的 `minimumReleaseAgeExclude` 对**校验阶段**不一定生效（它能放行 `--lockfile-only` 的解析）。
  等一两天（超过策略窗口）或临时放宽该策略即可。
* OAuth 桥是「每个服务器一行」，**套件不预置任何服务器行**（`1.3.1` 起）：装完套件不会多出任何 MCP 服务器，
  要用 Cloudflare 或别的服务器，就在「设置 → MCP」面板点“新增”，或在 profile 的 `insert:` 里自己写一条。
  `@hualiong/dsh-mcp-manager` 自己不带任何服务器配置（它的 README 里出现 `oauth-mcp-cloudflare` 只是 `credentialRef` 推导示例）。
* `cost-lite` 行默认关闭，计费徽标是 opt-in；它的价目只按精确 `(provider, model)` 命中，未登记组合整条跳过。
* 组件的 UI 不再共享一个浏览器产物，但 `dsh.client` 的排序只作用于单个包内部：跨组件的加载顺序由各自的 `inject` 决定。
* 组件内部的日志前缀沿用各组件包自己的前缀（`dsh-mcp-manager(...)` 等）。
* `@hualiong/dsh-mcp-manager` 的 `lib/` 里只有 `index.js` / `client.js` 与两个 sourcemap，没有 `lib/types/`，
  因此它**不再对外声明类型**（`0.1.6` 起删掉了指向不存在文件的 `types` / `exports[].types`，而不是留一个坏指针）。
  运行时不受影响（DSH Loader 只按 ESM import `lib/index.js` / `lib/client.js`）；要让 TypeScript 使用者拿到声明，
  得先让 `scripts/build.mjs` 产出 `.d.ts`（其 `tsconfig.json` 的 `paths` 指向 DSH checkout，`tsc --emitDeclarationOnly` 目前无法直接跑通）。
