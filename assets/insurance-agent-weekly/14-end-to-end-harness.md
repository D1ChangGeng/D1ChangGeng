# 一条请求穿过整个 Harness：从条款查询到持久事件

现在有一个明确需求：“请核对示例保障甲的等待期，并说明意外情形是否适用相同规则。”为了完成它，系统可能先检索定义，再读取来源页，最后给出带依据的回答。

这段业务过程背后，有一条可以具体观察的工程链：HTTP 接收输入，数据库建立运行对象，Worker 领取任务，SDK 调用模型，工具获取证据，结果进入会话，最终内容与运行终态共同提交，再通过事件接口输出。

本篇把 Insurance Agent Runtime 的这些模块连接成一次端到端阅读与验证。我们用合成条款追踪请求、模型、工具和事件之间的数据关系，观察一套通用 Harness（智能体执行控制系统）如何完成具体的软件工程任务。

## 先给这次运行一个可检查的目标

对于普通用户，目标是得到正确答案。对于运行系统，还要检查执行关系是否成立：问题只受理一次，工具按当前权限执行，下一次模型调用读取了已保存的结果，最终引用指向已提交内容，外部事件能够回溯全过程。

可以准备两种实验模式。

**服务生命周期观察**使用现有合成配置。它能够验证 HTTP、数据库、队列、运行终态和事件。默认固定合成模型可能直接回答，工具数量为零也是该模式下的正常结果。

**工具循环观察**使用会输出工具调用的受控模型夹具，或已授权的真实模型。它进一步经过 ToolInvocation（工具执行实例）、业务后端和工具结果回流。两种模式共享运行系统，但验证范围不同。[^runner]

因此，我们先阅读一条带工具的教学轨迹，再用一个最小外部观察程序连接实际受控实例。

## 第一站：请求成为持久工作

调用方先创建 Thread（对话线程），再提交 Turn（用户回合）。受理输入包括问题、Profile（运行配置名称）、线程版本和 Idempotency-Key（幂等命令标识）。

`RuntimeStorage.submitTurn()` 验证当前主体和资源归属，处理幂等重放，锁定线程，检查活动运行与版本，再解析可信运行快照。

同一事务保存用户条目、Turn、Run（执行尝试）、快照、事件、队列任务和回执。成功返回 `202` 后，调用方得到稳定的 `threadId`、`turnId`、`runId` 和 `eventsPath`。[^admission]

```text
外部输入
  → 用户条目 I1
  → Turn U1
  → Run R1，状态 queued
  → Job J1，数据 { runId: R1 }
  → turn.created / run.queued
```

这里的字母编号是教学标识。实际 ID 由服务端生成，不应从用户正文推导。相同幂等命令重放时复用原回执；重新提出同样文字但使用新命令，可能是新的业务回合。

## 第二站：Worker 取得执行资格

队列向 `RunWorker` 交付 Job，Worker 调用 `claimRun()` 领取 R1。数据库确认运行可以开始后，建立 Lease（执行租约），保存 owner（持有者）、fence（资格代次）和过期时间，并提交 `run.started`。[^worker]

`RunnerFacade.execute()` 建立本次 RunEnvironment（运行环境），装配模型、工具、Session（模型会话历史）和取消信号。这里开始使用已经冻结的配置，而不是重新相信 HTTP 正文中的任意运行参数。

```text
R1 queued
  → claimRun
  → R1 running + 有效租约
  → RunnerFacade
  → SDK Agent、Model、Tools、Session
```

后续写入仍然检查租约。执行器即使还活着，只要已经失去合法资格，就不能继续覆盖产品状态。

## 第三站：模型提出一个可执行意图

`RecordedModel` 在模型请求前检查预算和当前授权，创建 ModelStep（模型调用步骤）。模型第一次响应提出 `search_documents` 调用，参数包括查询内容、筛选条件和召回数量。

模型响应完成时，S1 可以结束。工具尚未执行，但调用意图已经成为保存的模型输出条目。`callId` 将这个意图连接到后续结果。[^model]

```text
ModelStep S1
  input：用户问题与当前可用工具
  output：search_documents 调用 C1
  status：completed
```

一次步骤对应模型边界上的调用尝试。实际网络请求和费用还需要传输与用量记录核对；这份教学轨迹只表达执行关联。

## 第四站：工具执行并把结果变成事实

`createRunTools()` 确认调用来自当前已记录的 ModelStep，检查工具名、版本、权限和预算，随后创建 V1。Registry（工具注册系统）再次确认当前授权和后端状态，把业务参数交给对应 Port（接口契约）的 Adapter（实际实现）。[^tools]

返回结果通过 Schema（结构约束）、数据来源和体积检查后，SDK 的结果提取回调取得确切工具结果条目。存储将该条目、V1 的成功状态和完成事件共同提交。

```text
调用 C1
  → ToolInvocation V1 running
  → 业务检索返回
  → 结果条目 E1 committed
  → V1 succeeded
  → tool_invocation.completed
  → SDK 可以消费 E1
```

这条先后关系是全链的关键。如果 E1 保存失败，第二次模型调用就不应继续依赖它。返回对象和已提交事实之间的区别，在异常恢复时会直接决定系统是否能够重建过程。

## 第五站：证据推动下一次动作

Session 将原问题、调用 C1 和结果 E1 组成后续模型输入。模型可能发现需要原文完整页，于是再次调用 `get_source_page`，传入 E1 中精确的 `sourceRef`（授权来源引用）。

来源工具解析引用，检查当前主体、文档版本和允许页面，再返回标准化来源内容。第二个工具结果提交后，模型取得更完整的证据，并在下一次调用中生成答案。

```text
S1 → C1 检索 → V1 / E1
S2 → C2 来源读取 → V2 / E2
S3 → 最终助手内容 A1
```

这是一条合理的教学路径，不是每个问题都必须执行的固定序列。真实模型可以直接回答、改变查询或使用其他可用工具；Harness 需要保证每条实际路径都遵守同一套身份、权限和提交规则。

来源存在也不自动证明答案语义正确。最终业务判断仍要核验条款是否支持相应结论。运行系统提供的是准确保存和回溯这份依据的基础。

## 第六站：提交最终引用与外部事件

模型输出保存为助手条目 A1。门面等待 SDK 完成，重新检查取消状态和租约，再调用 `finishRun()`。

存储确认 A1 是当前回合合法的已提交内容，写入 `Turn.finalContentRef`，更新 Run 终态，并保存 `turn.completed` 和 `run.completed` 事件。[^terminal]

```text
Turn U1.finalContentRef → A1
Run R1.status = completed
事件 turn.completed → A1
事件 run.completed → A1
```

SSE（服务器发送事件）读取的是已提交事件序列。它把 Thread 内的序号写入 `id` 字段，以事件类型和 JSON 负载对外输出。调用方可以在断线后按游标续接；观察连接的关闭不会自动取消后台 Run。

至此，业务请求已经从入口穿过模型与工具循环，成为可查询内容和可重放的运行事件。

## 用一个外部程序观察实际服务

下面的 Python 程序只使用标准库，连接已有的本地测试实例，创建 Thread、提交 Turn、读取 SSE，最后核对运行和回合状态。它会产生新的测试会话；使用前应选择已授权的测试 Profile，并提供该实例的本地令牌。

程序不建立模型配置，也不把默认合成输出误判成工具循环。若要观察工具调用，应使用前面说明的受控工具模型或已授权真实配置。

```python
import json
import os
import socket
import time
import uuid
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import (
    HTTPRedirectHandler, ProxyHandler, Request, build_opener,
)


class NoRedirect(HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        return None


base = os.environ.get('HARNESS_BASE_URL', 'http://127.0.0.1:3100').rstrip('/')
token = os.environ['HARNESS_TEST_TOKEN']
profile = os.environ['HARNESS_TEST_PROFILE']
parts = urlsplit(base)
if (parts.scheme not in {'http', 'https'}
        or parts.hostname not in {'127.0.0.1', 'localhost', '::1'}
        or parts.username or parts.password or parts.query or parts.fragment
        or parts.path not in {'', '/'}):
    raise ValueError('请使用本地受控 Harness 的根地址')

opener = build_opener(ProxyHandler({}), NoRedirect())


def make_request(method, path, body=None, command_key=None):
    if not path.startswith('/') or path.startswith('//'):
        raise ValueError('请求路径必须是本地 API 相对路径')
    headers = {'Authorization': 'Bearer ' + token}
    data = None
    if body is not None:
        data = json.dumps(body, ensure_ascii=False).encode('utf-8')
        headers['Content-Type'] = 'application/json'
    if command_key is not None:
        headers['Idempotency-Key'] = command_key
    return Request(base + path, data=data, headers=headers, method=method)


def request_json(method, path, body=None, command_key=None):
    request = make_request(method, path, body, command_key)
    with opener.open(request, timeout=10) as response:
        return json.load(response)


thread = request_json(
    'POST', '/v1/threads', {'title': '合成条款执行观察'}, str(uuid.uuid4()),
)
accepted = request_json(
    'POST', f"/v1/threads/{thread['threadId']}/turns",
    {
        'input': {'text': '请核对示例保障甲的等待期，并说明证据来源。'},
        'agentProfileId': profile,
        'expectedThreadRevision': thread['revision'],
    },
    str(uuid.uuid4()),
)
run_id = accepted['runId']
events = []
last_seq = -1
limit = time.monotonic() + 30
stream_request = make_request('GET', accepted['eventsPath'] + '?after=0')
stream_request.add_header('Accept', 'text/event-stream')

try:
    with opener.open(stream_request, timeout=5) as response:
        data_lines = []
        while time.monotonic() < limit:
            raw = response.readline()
            if not raw:
                break
            line = raw.decode('utf-8').rstrip('\r\n')
            if line.startswith(':'):
                continue
            if line.startswith('data:'):
                data_lines.append(line[5:].lstrip(' '))
                continue
            if line or not data_lines:
                continue
            event = json.loads('\n'.join(data_lines))
            data_lines = []
            seq = int(event['seq'])
            if seq <= last_seq:
                raise AssertionError('当前观察连接的事件顺序异常')
            last_seq = seq
            events.append(event)
            if event['runId'] == run_id and event['type'] in {
                'run.completed', 'run.failed', 'run.cancelled', 'run.interrupted',
            }:
                break
except (TimeoutError, socket.timeout):
    print('观察窗口结束；后台 Run 仍按自己的预算运行。')
except HTTPError as exc:
    raise RuntimeError(f'事件接口返回 HTTP {exc.code}') from None

run = request_json('GET', f'/v1/runs/{run_id}')
turn = request_json(
    'GET',
    f"/v1/threads/{accepted['threadId']}/turns/{accepted['turnId']}",
)
completed_events = [event for event in events
                    if event['runId'] == run_id and event['type'] == 'run.completed']
if completed_events:
    assert run['status'] == turn['status'] == 'completed'
    assert turn['finalContentRef'] == completed_events[-1]['payload']['finalContentRef']
    assert turn['finalContentRef']

print(json.dumps({
    'runId': run_id,
    'status': run['status'],
    'observedEvents': len(events),
    'observedModelSteps': sum(event['type'] == 'model_step.started' for event in events),
    'observedToolInvocations': sum(event['type'] == 'tool_invocation.started' for event in events),
    'finalContentRef': turn.get('finalContentRef'),
}, ensure_ascii=False, indent=2))
```

环境变量只需指定实例地址、测试令牌和已开放 Profile。脚本不打印令牌，也不跟随重定向。30 秒是这个观察程序的窗口，5 秒是一次流读取的空闲超时；它们都不会修改后台 Run 的执行预算。

如果观察结束时状态仍为 running，输出表示“本次只观察到部分过程”。模型和工具计数也明确标为 observed（当前观察所见），不能据此代替完整用量统计。

这个程序检查外部链路。要确认工具结果在下一次模型调用前已经提交，还应结合数据库和受控模型输入断言。

## 再运行一次可重复的故障场景

完整演示至少应加入一个有明确预期的异常。最有价值的选择之一，是在工具返回后、结果提交处安排失败。预期下一次模型请求保持未发生，并由运行系统记录失败或待收敛状态。

也可以让模型在完成边界等待，先提交取消，再放行完成事件，检查取消意图与成功提交的竞争由存储规则处理。

这些场景应使用项目受控测试夹具和隔离数据库。仓库提供 `test:runner`、`test:control` 和 `test:recovery` 等脚本，实际所需配置由对应测试入口确认。默认跳过需要数据库的测试组，不能当作真实全链验证。[^tests]

异常演示不追求制造更复杂的故障，而是解释一个清楚的保证：哪种已发生事实被保留，哪种后续动作被阻止，以及恢复从哪里继续。

## 把运行结果与业务质量一起阅读

最后得到的报告至少包含两部分。

工程部分检查请求受理、调用身份、工具结果、历史引用、终态和事件是否符合协议。业务部分检查原始证据是否支持等待期和意外情形的结论，以及相比固定检索增加了多少有效覆盖和成本。

一个结果可能工程上完全正确，但真实资料仍缺少答案；也可能模型偶然答对，却存在未保存工具结果的运行缺陷。把两类判断分开，才能知道下一步应该改善资料、模型策略，还是修复 Harness。

## 从自然语言任务回到明确的系统责任

一次咨询看起来只有一句话，实际却由多个可以独立解释的边界协作完成：控制面接纳命令，存储建立身份，Worker 管理执行资格，模型选择动作，工具边界提交结果，Session 构造连续输入，终态事务固定结论，事件协议对外表达发生过什么。

这就是完整 Harness 的产品价值：让模型能力进入一套可控制、可追踪、可恢复和可验证的软件系统。读者可以从任何一个实际问题切入，再沿这条链定位所需模块，而不必先记住一整套抽象名词。

继续查阅：[请求与事务](03-request-admission-and-transactions.md)、[工具执行生命周期](07-tool-invocation-lifecycle.md)、[事件输出协议](11-durable-events-and-streaming.md)。

## 实现依据

[^admission]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/transport/http/app.ts`、`storage/store.ts` 的 `submitTurn`，以及 `tests/control/http.test.ts` 的请求形状与受理验证。
[^worker]: 同一快照：`apps/runtime/src/runner/worker.ts`、`facade.ts` 与存储 `claimRun`。
[^model]: 同一快照：`apps/runtime/src/runner/model.ts`，ModelStep 和规范化模型输出记录。
[^tools]: 同一快照：`apps/runtime/src/runner/tools.ts`、`registry/registry.ts`、`business/ports.ts` 与业务 Adapter。教学轨迹是合成示例，不是重新运行真实模型取得的日志。
[^terminal]: 同一快照：`RuntimeStorage.finishRun`、`packages/contracts/src/events.ts` 和 `apps/runtime/src/transport/http/sse.ts`。
[^runner]: `docs/implementation/registry-runtime-integration.md` 与 `runner-preparation.md` 对默认合成输出、受控工具循环和真实模型验证范围的说明。
[^tests]: 同一快照根目录 `package.json`、`tests/runner/`、`tests/control/`、`tests/recovery/`。本文外部观察脚本使用产品公开 HTTP 契约，不能替代这些内部执行断言。
