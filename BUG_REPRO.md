# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

事件投递批次的第一条消息格式不受支持，让它进入重试没有问题；异常在于后面的合法推理事件也被留到下一次 worker tick，当前批次没有继续执行。测试文件和断言不要调整，任何相关用例都不能跳过或降低覆盖。请解决批次内的失败隔离，让坏消息按自身次数退避，其余任务仍在本轮完成。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-18
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-18.git
- parent SHA：8269406374866ba2017fdc2a9ab5858d1ec72eb9

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-18.git bug-repro
cd bug-repro
git checkout --detach 8269406374866ba2017fdc2a9ab5858d1ec72eb9
go test ./internal/worker -run ^TestPoisonJobDoesNotStopLaterJobsInBatch$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/worker -run ^TestPoisonJobDoesNotStopLaterJobsInBatch$ -count=1
2026/08/21 02:31:13 WARN outbox job failed job_id=job_invalid attempt=1 dead=false error="unsupported job kind \"unsupported_event\""
--- FAIL: TestPoisonJobDoesNotStopLaterJobsInBatch (0.00s)
    annotation_repo_behavior_test.go:63: later valid job status = "running", want "succeeded"
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/worker	0.014s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/worker -run ^TestPoisonJobDoesNotStopLaterJobsInBatch$ -count=1
2026/08/21 02:41:16 WARN outbox job failed job_id=job_invalid attempt=1 dead=false error="unsupported job kind \"unsupported_event\""
--- FAIL: TestPoisonJobDoesNotStopLaterJobsInBatch (0.02s)
    annotation_repo_behavior_test.go:63: later valid job status = "running", want "succeeded"
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/worker	0.304s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

同一领取批次中，第一条不受支持的消息只能按自身 attempts 进入失败重试，不得中止后续任务；后面的合法推理事件必须在当前 worker tick 内完成。定向批次隔离用例及相关 worker 回归须通过，现有测试、断言和覆盖范围不得修改或规避。
