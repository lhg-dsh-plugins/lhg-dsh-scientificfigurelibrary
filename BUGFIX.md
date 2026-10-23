# BUGFIX — 0.9.1 → 0.9.2

## 标题

资产指纹排序使用与语言环境相关的 `localeCompare`：同一资产集在不同 `LANG`
环境下算出不同的持久化指纹（`AssetFingerprintsV1` 集合哈希与 `contentHash`），
图库换语言环境打开时会被误判「registry component fingerprints do not match」，
且基于 `fullAssetSha256` 的别名/旧版匹配失效。

## 根因

`src/importers.ts` 中进入持久化哈希输入的两处排序使用了无 locale 参数的
`String.prototype.localeCompare`，其结果取决于进程默认 locale（随 `LANG`/
`LC_ALL` 与 ICU 默认值变化）：

- `descriptorSetHash()`（`assetFingerprints()` 的集合哈希）：按 `role`、`file`
  以 `localeCompare` 排序后 `JSON.stringify` 再 sha256。
- `contentHash()`：`code` / `references` 数组按 `a.file.localeCompare(b.file)`
  排序后进入 canonicalize 后的 JSON 哈希；`prepareDirectImport()` 中喂给
  `contentHash` 的 `code` 数组本身也按 `localeCompare` 预排序。

这与仓库自有规范矛盾：`src/canonical-json.ts` 专门提供 locale 无关的
`compareCanonicalStrings`（UTF-16 代码点序）并注释强调「unlike localeCompare
it is locale-independent」；其他内容寻址路径（`provider-source-fetch.ts`、
`provider-sources.ts`、`public-catalog-provider.ts` 等）则一律显式传 `"en"`
固定 locale。该两处是喂入持久化身份哈希的漏网排序。

## 证据链（修复前）

最小复现（离线纯函数，构造含排序敏感文件名的同一资产集，文件名为
`code/中.R`、`code/b.R`、`code/Z.R`、`code/ä.R`）：

```
LANG=en_US.UTF-8 node repro.mjs
{"locale":"en-US","executableCodeSetSha256":"23c704dbac38ab23a4e79f953d244f50d76ff47887031b79b53c5a54a8012da5", ...}
LANG=zh_CN.UTF-8 node repro.mjs
{"locale":"zh-CN","executableCodeSetSha256":"759bc7d98b83bcdaac965484400817f5109b8e87838c4e7e004c6820f92b441a", ...}
LANG=sv_SE.UTF-8 node repro.mjs
{"locale":"sv-SE","executableCodeSetSha256":"1fd0bd5d35170267ee782a2b6e90bce27f794cb03951ece5f9973fead0e9dbbc", ...}
```

同一资产集、仅语言环境不同，指纹三值互异。`git stash` 回基线复跑结果逐字
一致（en 23c704… / zh 759bc7…），证明差异来自基线代码。

影响面（代码路径）：`src/user-library.ts` 在打开图库时重算
`assetFingerprints()` 并与 registry 中持久化的指纹做 JSON 比对，不一致即报
`registry component fingerprints do not match the verified stored files`；
别名与旧版匹配（同文件）也直接比较 `fullAssetSha256` 是否相等。科研用户的
代码文件名常含中文（如「中.R」正是排序敏感例），在中文环境导入、英文环境
打开（或反之）即触发误报。

上游 open issue 核对（2026-10-23）：仅 3 条，均为功能请求（#66、#58、#22），
无此缺陷报告；本缺陷由静态审查定位、复现坐实。

## 改动文件

仅 `src/importers.ts` 一处逻辑文件（6 处插入、4 处删除）：

- 新增导入 `compareCanonicalStrings`（`./canonical-json.ts`）。
- `descriptorSetHash()`：`role`、`file` 排序改用 `compareCanonicalStrings`。
- `contentHash()`：`code`、`references` 的 `file` 排序改用 `compareCanonicalStrings`。
- `prepareDirectImport()`：`code` 预排序改用 `compareCanonicalStrings`。

其余为版本/出处动作：`package.json`（version 0.9.1→0.9.2、repository.url 改
落点仓）、用仓库官方 `npm run version:set -- 0.9.2` 同步的版本戳
（`package-lock.json` 两处、四宿主 `plugin.json`、`skills/figure-library/SKILL.md`
标题、`docs/PROTOCOL.md` 包名示例）、README.md 顶部追加 fork 头（原文未删）、
新增 UPSTREAM.md / BUGFIX.md。LICENSE 原样保留。

## 复验结果

- 复现脚本修复后：en-US / zh-CN / sv-SE 三种 locale 输出同一指纹
  `89a35ed929129ab59a8eda32d99d87376b329608ca57e80a189a4d0445ab9034`。
  （与旧 en-US 值也不同——代码点序与 en 排序本就不一致，见验证边界。）
- `npx tsc --noEmit` 退出 0。
- 定向测试：`node --test tests/imports.test.ts tests/canonical-json.test.ts`
  9/9 通过；`LANG=zh_CN.UTF-8 node --test tests/imports.test.ts` 7/7 通过。
- 全量 `npm test`：基线（上游 HEAD）退出 0；修复后（改版本号前）退出 0，
  零失败，无回归。
- `npm run version:check` 通过。版本/仓库改指后复跑门禁定向集
  （version-scripts / plugin-manifests / package-release-gates / dsh-plugin /
  imports / canonical-json，共 26 条）：25 过、1 条预期失败——
  plugin-manifests 的「host plugin manifests share version, skill, and MCP
  identity」把 `package.json` 的 `repository.url` 钉死为上游地址
  （`https://github.com/xuzhougeng/ScientificFigureLibrary`），是上游的身份
  守卫断言，fork 按契约改指落点仓后必然触发；其余版本一致性断言已逐项
  核对通过。本版未迁就修改该测试。

## 验证边界

- 未做真实宿主（Claude/Codex/dsh 等）端到端验收；结论建立在纯函数复现 +
  全量测试与基线一致之上。
- **存量指纹迁移影响（如实记录）**：旧版本在任一 locale 下生成的指纹都与
  修复后的代码点序指纹不一致——旧值本身随环境漂移、不构成稳定身份。升级后
  首次以新代码重算存量 registry 指纹时会出现一次不匹配报错，需要按图库的
  既有流程重新登记/导入使指纹以新口径落盘；本版未改 `algorithm`
  标识（`figure-library.asset-fingerprints.v1`），因为改名会让
  `validFingerprints` 直接拒绝全部存量 registry，影响更大；是否升算法版本
  应由上游决定。
- 本次仅修复进入持久化哈希的排序。同文件目录遍历顺序（如
  `walkGalleryCode` 的条目排序）不进入哈希、且结果与顺序无关，未动；
  其他模块展示用途的 `localeCompare` 不在本次范围。
