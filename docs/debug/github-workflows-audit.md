# `.github` 目录核查与个人分支管理调整报告

- **核查对象**：`.github/` 全量资产（共 21 个文件：12 个 Workflows + 9 个仓库治理/模板配置）
- **当前仓库上下文**：`origin` 指向个人 Fork 仓库 `https://github.com/TREYWANGCQU/owncloud-core.git`，当前分支 `master`
- **核查目标**：
  1. 盘点 `.github` 目录全部内容及其触发机制与依赖边界；
  2. 白盒核查邮件报错 `Changelog / changelog / Generate Changelog (Failed in 31 seconds)` 的触发链条；
  3. 给出面向个人分支管理的删除、保留与调整建议（兼顾“独立维护”与“未来同步上游”两种分支策略）。

---

## STATUS

`FAIL`

> 说明：`changelog.yml` 在个人 Fork 的 `master` 推送事件上存在确定性的 Secrets 缺失与上游工作流耦合冲突；同时 `php-unit.yml`、`acceptance.yml`、`translation-sync.yml` 与 `dependabot.yml` 均存在面向 `owncloud/core` 上游组织环境的硬编码依赖，在个人分支管理场景下会触发报错、无效校验或 CI 检出失败。

---

## VERIFIED ITEMS

### 1. 报错直接关联链核查：`Changelog / changelog / Generate Changelog`

- **对应文件**：[changelog.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/changelog.yml#L1-L16)
- **触发条件**：`on.push.branches: [master]`（每次向 `master` 推送或同步上游提交均会触发）。
- **执行链路**：调用上游复用工作流 `owncloud/reusable-workflows/.github/workflows/calens.yml@main`，并显式透传两个仓库级 Secrets：
  - `APP_ID: ${{ secrets.CALENS_APP_ID }}`
  - `APP_PRIVATE_KEY: ${{ secrets.CALENS_APP_PRIVATE_KEY }}`
- **失败机理判定**：该工作流用于 `owncloud` 官方组织内通过专属 GitHub App 自动生成并回写 `CHANGELOG.md`。在个人 Fork 仓库 `TREYWANGCQU/owncloud-core` 中，若未配置上述 `CALENS_APP_ID` / `CALENS_APP_PRIVATE_KEY`，工作流在尝试以 GitHub App 身份鉴权或提交时会在约数十秒内失败，并触发 GitHub Actions 邮件告警。

### 2. `.github/workflows/` 全量盘点（12 个工作流）

| 文件 | 名称 / 用途 | 触发器 | 外部依赖 / Secrets / Fork 兼容性核查 | 个人分支管理建议 |
| :--- | :--- | :--- | :--- | :--- |
| [changelog.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/changelog.yml) | Changelog 自动生成 | `push` (`master`) | 依赖 `owncloud/reusable-workflows/.../calens.yml@main` 及 `CALENS_APP_ID`、`CALENS_APP_PRIVATE_KEY` | **优先删除或禁用**（当前邮件报错直接来源） |
| [translation-sync.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/translation-sync.yml) | Transifex 翻译每日同步 | `schedule` (每日 02:00 UTC)、`workflow_dispatch` | 依赖 `TX_TOKEN`、`TRANSLATION_APP_ID`、`TRANSLATION_APP_PRIVATE_KEY` | **建议删除或禁用**（无官方密钥必败） |
| [nightly.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/nightly.yml) | Oracle 23ai 每日单测 | `schedule` (每日 02:30 UTC)、`workflow_dispatch` | 调用本地 `php-unit.yml`，拉取 `gvenzl/oracle-free:23-slim-faststart`；且受 `php-unit.yml#L73` 硬编码影响 | **建议删除或禁用**（重资源且依赖上游硬编码） |
| [stale.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/stale.yml) | 自动标记/关闭陈旧 Issue | `schedule` (每日 01:11 UTC) | `actions/stale`，需要 `issues: write` | **建议删除或禁用**（个人仓库通常无需自动清理社区 Issue） |
| [ci.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/ci.yml) | 主 CI 编排（Commits、JS/PHP 单测、Calens 校验、API/CLI/S3 验收） | `push` (`master`)、`pull_request` | 编排 `semantic-git-message`、`js-unit`、`php-unit`（含 Oracle）、`calens`、`acceptance`；每次推送消耗大量 Actions 额度且对提交规范/changelog 强校验 | **按需大幅精简或禁用**（若仅自用可删除/禁用；若需跑自测须先修复子工作流硬编码） |
| [php-unit.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/php-unit.yml) | PHP 单元测试（可复用子工作流） | `workflow_call` | **硬编码缺陷**：第 73 行写死 `repository: owncloud/core`；在个人 Fork 提交新 SHA 时将无法在上游仓库检出对应提交 | **若保留 CI 必须调整**（移除 `repository: owncloud/core`）；否则随 `ci.yml` 一并删除 |
| [acceptance.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/acceptance.yml) | API / CLI / WebUI 验收测试（可复用子工作流） | `workflow_call` | **硬编码缺陷**：第 137 行写死 `repository: owncloud/core`；另检出 `owncloud/testing`、`owncloud/files_primary_s3` 等公开仓库 | **若保留 CI 必须调整**（移除第 137 行硬编码）；个人分支通常成本过高，建议随 `ci.yml` 删除 |
| [js-unit.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/js-unit.yml) | JS 单元测试（可复用子工作流） | `workflow_call` | 使用默认 `actions/checkout`（无硬编码仓库），安装 Firefox 执行 `make test-js` | 仅由 `ci.yml` 调用；若停用 `ci.yml` 可一并删除，否则可保留 |
| [lint-and-codestyle.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/lint-and-codestyle.yml) | PHP 代码风格 / PHPStan / Phan | `push` (`master`)、`pull_request` | 无自定义 Secrets，无硬编码仓库，纯本地 `make` 检查 | **推荐保留**（或改为仅 `pull_request` / `workflow_dispatch` 触发以节省额度） |
| [lint-pr-title.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/lint-pr-title.yml) | PR 标题 Conventional Commits 校验 | `pull_request` | 仅使用内置 `GITHUB_TOKEN`，轻量 | **按需保留或删除**（个人自合 PR 若不想受标题格式限制可删） |
| [security-scan.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/security-scan.yml) | Trivy 依赖/构建物安全扫描 | `push` (`master`)、`pull_request`、`schedule` (每周一)、`workflow_dispatch` | 调用 `owncloud/reusable-workflows/.github/workflows/trivy.yml@main`，需完整执行 `make dist-dir` | **建议调整触发器或删除**（每周定时 + 每次推送均构建分发包，开销较大） |
| [zizmor.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/zizmor.yml) | Actions 工作流安全静态审计 | `push` (`master`)、`pull_request`、`schedule` (每周一)、`workflow_dispatch` | 调用 `owncloud/reusable-workflows/.github/workflows/zizmor.yml@main` | **按需删除或改为仅 `workflow_dispatch`**（若清理了大部分工作流，审计意义降低） |

### 3. `.github/` 非工作流配置文件盘点（9 个文件）

| 文件 | 作用 | Fork 环境核查结论 | 个人分支管理建议 |
| :--- | :--- | :--- | :--- |
| [CODEOWNERS](file:///d:/offices/Github/owncloud-core/.github/CODEOWNERS) | 将全仓库代码所有者指向 `@owncloud/classic-maintainers` | 个人账号仓库下该组织 Team 无效，属于上游残留 | **建议删除**（或改为自己的 GitHub 用户名） |
| [dependabot.yml](file:///d:/offices/Github/owncloud-core/.github/dependabot.yml) | 配置 `composer`、`npm`、`github-actions` 共 6 组每周依赖更新 PR | 包含 3 组指向 `target-branch: "10.16"` 的配置，而当前 `origin` refs 中不存在 `10.16` 分支；且在个人仓库开启后会每周自动生成大量升级 PR | **建议删除或精简**（若不维护上游依赖升级，直接删除可避免 PR 噪音） |
| [config.yml](file:///d:/offices/Github/owncloud-core/.github/config.yml) | `behaviorbot/update-docs` 配置，提醒 PR 补充 `changelog/unreleased/` | 仅在仓库安装该 Probot App 时生效，未安装时为静态死文件 | **建议删除** |
| [lock.yml](file:///d:/offices/Github/owncloud-core/.github/lock.yml) | `dessant/lock-threads` 配置，365 天后锁定已关闭 Issue/PR | 同上，未安装对应 GitHub App 时不生效 | **建议删除** |
| [ISSUE_TEMPLATE/config.yml](file:///d:/offices/Github/owncloud-core/.github/ISSUE_TEMPLATE/config.yml) | 禁用空白 Issue，外链指向 `central.owncloud.org` 与 `owncloud.com/security/` | 在个人仓库中会阻止创建空白 Issue 并将人导流至官方站点 | **建议删除或替换** |
| [ISSUE_TEMPLATE/issue_template.md](file:///d:/offices/Github/owncloud-core/.github/ISSUE_TEMPLATE/issue_template.md) | ownCloud 官方 Bug 报告模板 | 仅影响新建 Issue 表单 | 按需删除或精简 |
| [PULL_REQUEST_TEMPLATE.md](file:///d:/offices/Github/owncloud-core/.github/PULL_REQUEST_TEMPLATE.md) | ownCloud 官方 PR 检查清单模板 | 仅影响新建 PR 描述框 | 按需删除或精简 |
| [CONTRIBUTING.md](file:///d:/offices/Github/owncloud-core/.github/CONTRIBUTING.md) | ownCloud 官方社区贡献指南 | 纯文档，无自动触发副作用 | 保留作为参考或删除均可 |
| [SECURITY.md](file:///d:/offices/Github/owncloud-core/.github/SECURITY.md) | ownCloud 官方安全漏洞上报指引 | 纯文档，无自动触发副作用 | 保留作为参考或删除均可 |

---

## ASSUMPTION MATRIX

- `[Verified Fact]` 本地仓库 `origin` 指向 `https://github.com/TREYWANGCQU/owncloud-core.git`，当前检出分支为 `master`。
- `[Verified Fact]` `.github/workflows/changelog.yml` 监听 `push` 到 `master`，调用 `owncloud/reusable-workflows/.github/workflows/calens.yml@main` 并传入 `secrets.CALENS_APP_ID` 与 `secrets.CALENS_APP_PRIVATE_KEY`。
- `[Verified Fact]` `.github/workflows/translation-sync.yml` 监听每日定时任务，传入 `secrets.TX_TOKEN`、`secrets.TRANSLATION_APP_ID`、`secrets.TRANSLATION_APP_PRIVATE_KEY`。
- `[Verified Fact]` `.github/workflows/php-unit.yml` 第 73 行与 `.github/workflows/acceptance.yml` 第 137 行均显式硬编码 `repository: owncloud/core`。
- `[Verified Fact]` `.github/dependabot.yml` 配置了 3 组 `target-branch: "10.16"`，而当前 `git show-ref` 中不存在 `10.16` 分支引用。
- `[Unverified Assumption]` 本次核查未拉取 GitHub Actions 线上运行日志，因此 `Failed in 31 seconds` 的具体终端报错文本（例如 `Input required and not supplied: app-id` 或 GitHub App Token 生成失败）未直接观测，但与其缺失 Secrets 的静态配置事实高度一致。
- `[Unverified Assumption]` 未确认 `TREYWANGCQU/owncloud-core` 的后续维护模式（是长期独立魔改维护，还是仍会定期 `rebase` / 合并上游 `owncloud/core` 甚至向上游提 PR），以及仓库级 Actions 开关与定时工作流是否已在 GitHub 网页端手动启用。

---

## POTENTIAL LOCAL OPTIMA & INQUIRIES

### 局部最优防卫：切勿盲目在 `master` 直接删除 `.github/` 而不评估同步策略

直接删除 `.github/` 下的文件虽然能立刻消除报错邮件，但存在明显的**路径依赖与维护策略分歧**：

1. **如果你未来还会从上游 `owncloud/core` 同步代码（`git pull --rebase` / `merge`）或向上游提 PR**：
   - 直接删除或修改 `.github/workflows/*.yml` 会导致上游每次更新工作流 SHA（Dependabot 每周都会更新）时产生持续的合并/变基冲突；
   - 若不慎将删除 `.github/` 的提交包含在发往上游的 PR 中，会破坏上游仓库 CI。
2. **如果你是将该仓库作为独立个人/内部定制分支长期维护，不打算向上游回馈 PR**：
   - 直接清理无关工作流与模板文件最干净，但要注意 `php-unit.yml` 和 `acceptance.yml` 中硬编码的 `repository: owncloud/core`（若你后续想在自己分支跑通 CI，仅删 `changelog.yml` 不修这两处会导致单测/验收继续失败）。

**请确认你的分支管理模式，以便决定采用哪一档实施方案**：
1. **零代码改动模式（推荐给需要持续同步上游的用户）**：不在 Git 中删除文件，直接在 GitHub 仓库页面 `Actions` 标签页逐个点击 `Disable workflow`（或直接关闭整个仓库的 GitHub Actions）。
2. **上游兼容守卫模式（折中方案）**：仅在需要屏蔽的工作流 Job 上追加 `if: github.repository == 'owncloud/core'`，保留文件结构以尽量降低冲突面积。
3. **独立分支彻底精简模式（推荐给独立二次开发/私有维护分支）**：按上表直接删除上游组织专属工作流与配置文件，仅保留轻量 `lint-and-codestyle.yml`（或按需修复 `php-unit.yml` 后的精简 CI）。

---

## WARNINGS

1. **隐藏的检出对象错位风险（`php-unit.yml#L73`、`acceptance.yml#L137`）**：两处 `actions/checkout` 均写死了 `repository: owncloud/core`。在个人 Fork 中触发 `ci.yml` 或 `nightly.yml` 时，若检出目标为个人 Fork 独有的 commit SHA，将导致检出失败或错测上游代码。
2. **定时任务潜在持续失败与额度消耗**：`translation-sync.yml`（缺 Transifex/App 密钥必败）、`nightly.yml`（每日拉起 Oracle 23ai 容器）、`security-scan.yml`（每周构建 `dist-dir`）在 Fork 若开启了定时触发，会持续消耗 Actions 分钟数并产生额外告警邮件。
3. **Dependabot 失效目标分支**：`dependabot.yml` 中指向 `target-branch: "10.16"` 的 3 组规则在当前远端无对应分支，可能产生无效检查告警。
4. **外部可复用工作流漂移风险**：`changelog.yml`、`translation-sync.yml`、`security-scan.yml`、`zizmor.yml`、`ci.yml` 均引用了 `@main` 可变分支（`owncloud/reusable-workflows/.../@main`），而非固定 commit SHA，上游工作流变更会直接影响下游仓库行为。

---

## ERRORS

1. **`Changelog / changelog / Generate Changelog` 必败缺陷**：[changelog.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/changelog.yml#L11-L16) 在个人 Fork 的 `master` 推送事件上无条件调用 `owncloud/reusable-workflows/.github/workflows/calens.yml@main`，且依赖官方组织私有 Secrets（`CALENS_APP_ID`、`CALENS_APP_PRIVATE_KEY`），导致每次推送 `master` 均触发失败告警邮件。
2. **`php-unit.yml` / `acceptance.yml` 跨仓库检出缺陷**：[php-unit.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/php-unit.yml#L70-L74) 与 [acceptance.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/acceptance.yml#L134-L138) 硬编码 `repository: owncloud/core`，在个人 Fork 提交新代码后触发 CI 时存在检出目标与触发仓库不一致的契约断裂。

---

## MINIMAL FIX SUGGESTIONS

*(严格遵循 Verifier 只读原则，以下仅提供最小变更建议，未直接修改仓库源码)*

### 方案 A：零 Git 冲突（推荐优先，适合仍需同步上游）
1. 打开 GitHub 仓库：`https://github.com/TREYWANGCQU/owncloud-core/actions`
2. 在左侧工作流列表中依次选中并点击右上角 `...` -> **Disable workflow**：
   - `Changelog`（立即止血当前邮件报错）
   - `Translation Sync`
   - `Nightly`
   - `Close stale issues`
   - （可选）若不需要自动跑重型测试，一并禁用 `CI`、`Security Scan`、`Zizmor`。

### 方案 B：独立分支精简清理（适合完全自主维护分支）
1. **直接删除以下上游组织专属文件**：
   - `.github/workflows/changelog.yml`
   - `.github/workflows/translation-sync.yml`
   - `.github/workflows/nightly.yml`
   - `.github/workflows/stale.yml`
   - `.github/CODEOWNERS`
   - `.github/config.yml`
   - `.github/lock.yml`
   - `.github/dependabot.yml`（或删除其中 `target-branch: "10.16"` 的 3 组配置）
   - `.github/ISSUE_TEMPLATE/config.yml`
2. **按需决定是否保留主 CI**：
   - 若**不需要**在 GitHub Actions 跑测试：一并删除 `ci.yml`、`acceptance.yml`、`php-unit.yml`、`js-unit.yml`、`security-scan.yml`、`zizmor.yml`、`lint-pr-title.yml`，仅保留 [lint-and-codestyle.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/lint-and-codestyle.yml) 即可。
   - 若**需要**在个人分支上跑 PHP 单元测试/验收测试：必须在 [php-unit.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/php-unit.yml#L73) 与 [acceptance.yml](file:///d:/offices/Github/owncloud-core/.github/workflows/acceptance.yml#L137) 中删除 `repository: owncloud/core` 行，使其默认检出当前触发仓库（`${{ github.repository }}`）。

### 方案 C：上游兼容的条件守卫（最小文件侵入）
若既希望保留 `.github/workflows/changelog.yml` 等文件以减少未来同步差异，又不希望在 GitHub UI 手动逐个禁用，可在对应 job 下补充仓库条件守卫：

```yaml
# .github/workflows/changelog.yml
jobs:
  changelog:
    if: github.repository == 'owncloud/core'
    uses: owncloud/reusable-workflows/.github/workflows/calens.yml@main
```
