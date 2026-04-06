# Folio PR #50 本地离线审查报告

- PR：#50 `feat/request-validation-alpha`
- 分支：`feat/request-validation-alpha`
- 本地 HEAD：`30c684b` (`feat: add request input and validation alpha baseline`)
- 审查方式：仅本地、只读、离线检查；未执行任何远端写操作
- 审查时间：2026-04-06

## 【PR 概要】

这次改动是一个**request input / validation 的 alpha 基线补丁**，总共 1 个 commit，涉及 6 个文件变更（3 个生产代码、3 个测试），主要新增了三块能力：

1. **Request 读取辅助能力**
   - 文件：`src/Core/Http/Request.php`
   - 新增方法：
     - `input(string $key, mixed $default = null): mixed`：从 query 取值
     - `header(string $key, mixed $default = null): mixed`：从 `$_SERVER` 风格数组取 header
     - `bodyInput(string $key, mixed $default = null): mixed`：从数组 body 取值

2. **验证器最小实现**
   - 文件：`src/Core/Validation/Validator.php`
   - 新增 `Validator::validate(array $data, array $rules): array`
   - 当前支持的规则只有：
     - `required`
     - `string`
     - `integer`
     - `array`
   - 校验失败时会聚合错误并抛出 `ValidationException`

3. **统一 JSON 错误边界接入验证异常**
   - 文件：`src/Core/Exceptions/ValidationException.php`
   - `ValidationException` 继承 `HttpException`
   - 固定错误语义：
     - HTTP 422
     - code = `VALIDATION_FAILED`
     - message = `The given data was invalid.`
     - `meta.errors` 中带字段级错误详情
   - `tests/Unit/Exceptions/HandlerTest.php` 补了针对统一错误出口的断言

### 变更文件清单

- `src/Core/Exceptions/ValidationException.php`（新增）
- `src/Core/Http/Request.php`（修改）
- `src/Core/Validation/Validator.php`（新增）
- `tests/Unit/Exceptions/HandlerTest.php`（修改）
- `tests/Unit/Http/RequestTest.php`（新增）
- `tests/Unit/Validation/ValidatorTest.php`（新增）

### 这次 PR 实际引入的契约

从代码本身看，这次 PR 更像是**底层能力打底**，而不是“请求验证已正式接入 HTTP 主链路”。当前能确认的契约是：

- 请求对象现在支持读取 query / body / route params / headers 的最小 helper
- 验证器现在支持最基础的字段必填和类型校验
- 校验失败时，可以通过 `ValidationException` 落到现有统一 JSON error boundary

但当前**没有看到这套能力已经接入 runtime / router / middleware / controller-style 入口**。我在仓库内 grep 后，生产代码里只有类定义本身，没有其他地方实例化 `Validator` 或把 request validation 挂入 HTTP 流程。也就是说：

- 这是一个**可被后续接入的 alpha 基线**
- 不是一个“用户现在已经能在主链路自动用起来”的完整功能

## 【代码与变更面审查】

### 1) Request 改动评估

`Request` 的三个新增 helper 方向是对的，补齐了最基本的读取能力。

但当前语义需要注意：

- `input()` **只读 query**，不读 body
- `bodyInput()` 只在 body 是数组时工作
- `header()` 依赖 `$_SERVER` 数组 key 的特定格式

这里最值得注意的是：**`input()` 这个命名很容易让人误解为“统一输入源”**，但实现上它只是 query helper。这个不是代码错误，但如果后续不补文档，很容易有人按 Laravel/常见 Web 框架习惯误用。

### 2) Validator 改动评估

`Validator` 当前实现很小，但结构清楚：

- 遍历字段规则
- 先处理 `required`
- 对存在且非 null 的值继续做类型校验
- 聚合错误后统一抛 `ValidationException`

这是一个合理的 alpha baseline。

不过它目前是**严格类型校验**：

- `integer` 必须真的是 PHP `int`
- query string 里的 `"18"` 不会被接受为 integer

这对“解码后的 JSON body”可能还行，但对 URL query 场景会比较苛刻。

### 3) 错误处理路径评估

这部分做得还算整齐：

- `ValidationException` 直接走现有 `HttpException` 系统
- `Handler` 会把 `meta.errors` 稳定放进统一 JSON error payload
- `HandlerTest` 覆盖了 422 错误边界输出

这说明作者至少把“异常出口口径统一”这件事想清楚了，不是胡乱扔个自定义异常完事。

## 【测试情况】

### 已运行命令

1. 仓库与分支确认

```bash
git status --short --branch
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
git diff --name-status main...HEAD
git diff --stat main...HEAD
git log --oneline main..HEAD
```

结果：
- 当前分支正确：`feat/request-validation-alpha`
- HEAD 正确：`30c684bfdba1d1f4dcd8503e62b6cd582393b45a`
- 与任务上下文一致
- 工作区有未跟踪文件：`merge_gate_result.json`

2. 单元测试

```bash
composer test
```

结果：**全部通过**

- PHPUnit 11.5.44
- PHP 8.3.20
- `54 tests, 160 assertions`
- 无失败、无错误

3. 代码风格检查（仓库脚本）

```bash
composer cs:check
```

结果：**脚本失败，但失败原因看起来是仓库级脚本/配置问题，不是本 PR 改动文件的直接风格问题**。

报错核心信息：

```text
You must call one of in() or append() methods before iterating over a Finder.
```

这说明当前 `php-cs-fixer` 是按默认配置启动，但没有提供有效路径/配置文件，导致脚本本身不可用。

4. 针对本 PR 改动文件的逐文件风格验证

我额外对本 PR 改动的 6 个文件逐个执行了 `php-cs-fixer --dry-run` 本地检查。

结果：**6 个文件均显示 `Found 0 of 1 files that can be fixed`**，即：

- `src/Core/Exceptions/ValidationException.php`
- `src/Core/Http/Request.php`
- `src/Core/Validation/Validator.php`
- `tests/Unit/Exceptions/HandlerTest.php`
- `tests/Unit/Http/RequestTest.php`
- `tests/Unit/Validation/ValidatorTest.php`

从逐文件结果看，本 PR 改动本身**没有明显代码风格问题**。

### 额外本地验证（手工打点）

除了测试外，我还做了两个本地 one-liner 行为验证：

1. **验证 `header('Content-Type')` 行为**

结果：返回默认值 `missing`，没有命中 `CONTENT_TYPE`。

这说明当前新增的 `Request::header()` 对常见 CGI 特殊头字段支持不完整。

2. **验证 query 中 `"18"` 走 `integer` 规则**

结果：抛出 `ValidationException`，错误为：

```php
['age' => ['The field must be an integer.']]
```

说明当前 validator 对 query 参数没有任何类型归一化/转换，完全按 PHP 原始值做严格校验。

## 【治理 / 合并门禁检查】

本地存在 `merge_gate_result.json`，内容摘要如下：

```json
{
  "gate": "eligible",
  "mergeable": true,
  "issue_refs": ["48", "49"],
  "labels": ["status/in-review", "risk/medium"],
  "checks": {
    "issue_linked": true,
    "status_ready": true,
    "blocked": false,
    "risk_allowed": true,
    "required_checks_ok": true,
    "review_decision": "",
    "mergeable_state": "unstable"
  },
  "reasons": []
}
```

### 解读

- 正向信号：
  - `gate = eligible`
  - `mergeable = true`
  - `required_checks_ok = true`
  - issue 已关联，未被 block，risk 允许

- 需要注意的地方：
  - `review_decision` 为空
  - `mergeable_state = unstable`

这个 JSON 从治理角度看，**没有给出硬阻断**，但也没有形成“审查已完成、状态很稳”的感觉。更像是：

- 机器门禁基本过了
- 但人工 review / 最终收口还没彻底闭环

## 【潜在风险点】

### 1. `Request::header()` 对常见标准头访问不完整

这是我本轮审查里最明确的真实问题。

当前实现：

```php
$normalized = 'HTTP_'.strtoupper(str_replace('-', '_', $key));
return $this->server[$normalized] ?? $this->server[strtoupper($key)] ?? $default;
```

这会导致：

- `header('X-Trace-Id')` 能正常命中 `HTTP_X_TRACE_ID`
- 但 `header('Content-Type')` 不会命中 `CONTENT_TYPE`
- 同理 `Content-Length` 这类 CGI 特殊字段也可能取不到

我本地复现结果就是：

```php
$r->header('Content-Type', 'missing') === 'missing'
```

这属于**新增 helper 的实际行为缺陷**，而且是相当常见的 header 场景，不只是边角料。

### 2. `input()` 命名存在误导性

现在的 `input()` 只读 query：

```php
return $this->query[$key] ?? $default;
```

如果后续开发者看到 `input('name')`，很容易默认它是“统一输入源（query + body）”。

这不是立即炸的 bug，但会形成**接口契约误解风险**。

### 3. validator 对 query 场景过于严格

当前 query 中的数值天然是字符串，例如：

- `?age=18` 实际上是 `"18"`

而 `integer` 规则要求必须是 PHP `int`。这意味着：

- 直接拿 `Request::input()` 的 query 结果去做 `integer` 校验，默认就会失败

如果这套能力后续要作为“request validation baseline”对外给人用，至少要：

- 在文档里说明这是**纯原始值校验**
- 或后续引入规范化 / casting / typed input 层

否则开发体验会比较拧巴。

### 4. 当前能力尚未接入主 HTTP 使用路径

生产代码搜索结果看，`Validator` 当前没有被 router / kernel / application / route handler 等主路径消费。

这意味着：

- 这 PR 提供的是“可用积木”
- 还不是“完整功能闭环”

如果团队对 PR 标题的预期只是“alpha baseline”，那没问题；
但如果有人按“validation feature 已可用”理解，就会有认知偏差。

### 5. 文档没有同步说明这块 alpha 能力

README 里目前仍写着：`request validation` 还不在 main 已承诺能力范围内。

而这个 PR 实际已经把 validation baseline 的一些底层能力放进来了，但没有同步说明：

- 现在到底新增了什么
- 哪些还没承诺稳定
- `input()` / `bodyInput()` 的语义分别是什么
- validator 当前支持哪些规则

这不构成代码 blocker，但会影响后续使用和评审预期。

## 【阻断因素 Blockers】

### Blocker 1：`Request::header()` 新增契约存在明确行为缺陷

我认为这是**当前最像真实 blocker 的点**。

理由：

- 这是 PR 新增的公开 helper
- 对常见 header 访问（如 `Content-Type`）会直接取值失败
- 属于 correctness 问题，不是风格或命名偏好问题
- 已被本地最小复现验证

如果这 helper 要作为 alpha 基线进主分支，我建议**至少先修掉这个点并补测试**。

### 非 blocker 但必须说明的前提

以下暂不判成硬 blocker，但合并时应明确知情：

- validator 目前还没有接入 HTTP 主链路，只是基础积木
- `input()` 只查 query，不是统一输入聚合器
- `integer` 规则对 query string 很严格，开发者易踩坑
- `composer cs:check` 现状不可用，属于仓库级脚本问题，和本 PR 关系不大，但会影响本地 merge 前检查体验

## 【合并准备度判断】

**结论：暂不建议直接合并。**

一句话判断：**技术上还不能算“放心可合并”，主要原因是这次新增的 `Request::header()` 在常见 `Content-Type` 场景下存在真实取值缺陷；在修复该问题并补齐对应测试后，才比较像“可考虑合并”的状态。**

如果团队愿意把这 PR 明确定位成“纯底层草稿 / baseline scaffold”，那其余问题基本都还能接受；但当前这个 header bug 不太适合带着进 main。

## 【后续推荐动作】

1. **优先修复 `Request::header()`**
   - 至少兼容：
     - `Content-Type` -> `CONTENT_TYPE`
     - `Content-Length` -> `CONTENT_LENGTH`
   - 并补充单元测试，而不是只测 `X-Trace-Id` 和 `CONTENT_TYPE` 常量式调用

2. **补充 request helper 的契约测试**
   - 建议新增测试覆盖：
     - `header('Content-Type')`
     - `header('Content-Length')`
     - body 非数组时 `bodyInput()` 默认值行为
     - query/body 同名键时的预期读取规则（哪怕现在不做聚合，也应该写清楚）

3. **明确 `input()` 的语义，必要时改名或补文档**
   - 如果它就是 query helper，名字可以更直白；
   - 如果打算以后成为统一输入入口，那现在至少要在文档或注释里明确“当前 alpha 仅支持 query”。

4. **明确 validator 的使用边界**
   - 文档说明当前仅支持：`required|string|integer|array`
   - 说明当前是严格类型校验，不会自动把 query string cast 成 int

5. **补一份简短 docs / changelog**
   - 说明这次只是 validation baseline，不代表主链路自动验证已经完成
   - 避免团队对能力完成度产生误解

6. **后续可以补一轮集成测试**
   - 例如给某个测试路由接上 `Validator`
   - 验证真实 HTTP 请求 -> 校验失败 -> 422 JSON error 的闭环
   - 这样才算从“有积木”走到“有主链路证明”

---

## 最终结论（给合并者的短版）

- **PR 做了什么**：补了 request helper、最小 validator、422 validation exception，以及对应单测。
- **测试情况**：`composer test` 全绿；仓库级 `composer cs:check` 失败，但看起来是既有脚本配置问题；改动文件本身逐文件 cs 检查无问题。
- **最大风险**：新增 `Request::header()` 对 `Content-Type` 这类常见头访问存在真实缺陷。
- **是否可考虑合并**：**目前不建议直接合并**；先修 header bug + 补测试，再谈合并会更稳。