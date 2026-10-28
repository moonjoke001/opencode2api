# opencode2api — patch 分支

> 本分支只放我们的线上补丁，**不是上游**。原始文档看 `main` 分支或上游
> `jasonxu114514/opencode2api`。

## 基线

上游 tag `v1.4.5` + 本地提交（`git log --oneline v1.4.5..patch` 可查）。

## 为什么打这些补丁

免费额度按项目 id × 出口记账，每天 00:00 UTC 重置（上游 issue #156）。
一个会话烧掉桶之后，剩下每个出口对同一身份都回 `429 Free usage limit
reached`——而换个新身份，任意出口都 200。box 上实测：被烧桶的固定会话
连跪 8 腿；全新会话首腿全进，8/8。

三个事实决定了设计方向：

- 游走起点按会话确定（FNV 亲和哈希），同会话永远从同一出口开始。
  “某出口恒 429”是亲和性，不是出口坏了。
- `#pool` 节点按设计永不冷却（`MarkFailure` 跳过，`Next()` 无视冷却直接
  返回），所以按代理名冷却等于没写。桶绑在**身份**上，不在出口上。
- 只换 project 不够，session 必须一起换（上游 #156 验证过：两个都换才
  从 0/8 恢复到 8/8）。

## 补丁清单

### 1. 配额 429 腿级轮换（sesrot1）

- `internal/gateway/abuse.go`：新增 `isFreeUsageLimit`——状态码 429 且正文
  带 "Free usage limit" 才算配额 429；边缘限流（空正文等）走老逻辑。
- `internal/gateway/upstream.go`（`doAnonymousUpstream`）：某腿吃到配额
  429，下一出口换全新会话（`CanonicalSessionID` + 随机信号，时间戳保持在
  上游接受窗口内）+ 全新项目 id 再试。
- 日志事件 `anonymous_session_rotated`。只动匿名通道。

### 2. 烧桶会话预轮换（sesrot2）

- `internal/gateway/gateway.go`：新增 `burnedSessions`——最近吃过配额 429
  的会话记 15 分钟（mutex 保护，上限 512 条，过期优先、满了随机淘汰）。
- `doAnonymousUpstream` 入口：带着烧桶会话的请求，首腿直接换新身份，
  不再花一腿去证明“桶还烧着”。日志事件
  `anonymous_session_prerotated`。
- 记录的是**吃到 429 的那个旧身份**（先记再换），内存永远指向真正烧掉
  的身份。健康会话不受影响，正常亲和性与提示词缓存保留。

## 构建与部署（box VPS 192.129.200.200）

```bash
docker build -t oc2:v1.4.5-sesrotN --build-arg VERSION=v1.4.5-sesrotN /root/oc2-patch
docker tag oc2:v1.4.5-sesrotN ghcr.io/jasonxu114514/opencode2api:v1.4.5
docker stop oc2 && docker rm oc2
docker compose -f /root/oc2/docker-compose.yml up -d
```

retag 是为了不改 `docker-compose.yml`（里面写的就是 `ghcr.io/...:v1.4.5`），
autoupdate 定时器继续判 already latest、无动作。下个上游 release 一拉取
就自然替换掉补丁镜像——这是设计好的交接。

回滚：补丁前镜像 `oc2:v1.4.5` 留在机器上，
`docker tag oc2:v1.4.5 ghcr.io/jasonxu114514/opencode2api:v1.4.5` + 重建即可。

## 验证

```bash
curl -s http://127.0.0.1:8086/healthz   # 版本 v1.4.5-sesrotN、keys、proxies
docker logs oc2 2>&1 | grep 'anonymous_session_prerotated'   # 预轮换在触发
docker logs oc2 2>&1 | grep 'anonymous_session_rotated'      # 腿轮换在触发
```

sesrot2 之后预期：带着烧桶会话的请求 1 腿结束（~3 秒），之前是 429 再
200 共 2 腿（~6 秒）；全新会话请求 1 腿结束。

## Rebase 约定

每个上游新 release：把 `patch` rebase 到新 tag，重构建、重部署、按上面
验证，sbc 部署档案（`b60ae47a…`）追加一笔。

## 相关

- 上游 issue #156 + PR #157（探针/Playground 身份问题）——互补不重叠：
  它们修探针和压测台，这两个补丁修线上流量。
- 只部署在 box VPS（`oc2` 容器），truenas 没动。
- box 源码树：`/root/oc2-patch`（`.bak-sesrotN` 是补丁前备份）。删掉的
  zen keys 在 vault 里（`oc2 zen key …`）。
