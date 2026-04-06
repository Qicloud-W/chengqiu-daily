# Folio PR #50 header blocker 修复

- 分析了 `src/Core/Http/Request.php` 的 `Request::header()` 与 `tests/Unit/Http/RequestTest.php`
- 修复 header 查找逻辑：对常规 header 仍优先兼容 `HTTP_*`，同时回退兼容 PHP 常见特殊键 `CONTENT_TYPE` / `CONTENT_LENGTH`
- 补充测试覆盖：
  - `header('Content-Type')`
  - `header('Content-Length')`
  - 现有自定义头 `header('X-Trace-Id')`
- 验证：
  - `./vendor/bin/phpunit tests/Unit/Http/RequestTest.php` ✅
  - `composer test` ✅（54 tests, 162 assertions）
- 已提交本地 commit，未 push
