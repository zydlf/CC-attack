# Java 服务抗 CC 防护方案（访问端口 / 登录端口 / 搜索端口）

> 我不能协助发起 CC/DDoS 攻击；以下内容用于**合法授权的防护与压测**。

## 1) 端口与接口分级策略

建议把你提到的三类入口拆成三种限流等级：

- **访问端口（普通页面/API）**：中等限流，保护总体吞吐。
- **登录端口（/login）**：严格限流 + 人机验证 + 失败惩罚。
- **搜索端口（/search）**：最容易被刷，按 IP + 用户 + 查询复杂度联合限流。

## 2) Nginx 网关限流（第一道）

```nginx
http {
    # 按真实客户端 IP 限流（有 CDN 时请配置 real_ip）
    limit_req_zone $binary_remote_addr zone=normal_zone:20m rate=20r/s;
    limit_req_zone $binary_remote_addr zone=login_zone:20m  rate=3r/s;
    limit_req_zone $binary_remote_addr zone=search_zone:20m rate=8r/s;

    # 并发连接数限制
    limit_conn_zone $binary_remote_addr zone=conn_zone:20m;

    server {
        listen 80;
        server_name your.domain;

        # 访问端口
        location /api/ {
            limit_req zone=normal_zone burst=40 nodelay;
            limit_conn conn_zone 50;
            proxy_pass http://java_backend;
        }

        # 登录端口
        location = /login {
            limit_req zone=login_zone burst=6 nodelay;
            limit_conn conn_zone 10;
            proxy_pass http://java_backend;
        }

        # 搜索端口
        location /search {
            limit_req zone=search_zone burst=12 nodelay;
            limit_conn conn_zone 20;
            proxy_pass http://java_backend;
        }
    }
}
```

## 3) Java 应用层限流（第二道）

推荐在 Spring Boot 使用 Bucket4j / Resilience4j。

- `/login`：按 `IP + username` 限制（例如 1 分钟 10 次）。
- `/search`：按 `IP + userId` 限制（例如 1 秒 5 次，1 分钟 100 次）。
- `/api`：按 token 或 API key 限制，避免单 IP 穿透。

伪代码：

```java
String key = ip + ":" + endpoint + ":" + userId;
if (!rateLimiter.tryConsume(key, 1)) {
    return ResponseEntity.status(429).body("Too Many Requests");
}
```

## 4) 登录端口专项防护

- 连续失败 5 次：触发验证码。
- 连续失败 10 次：账号或 IP 短时冻结（5~15 分钟）。
- 登录接口只允许 `POST`，拒绝异常 UA、空 Referer（按业务酌情）。
- 登录成功后签发短期 token，并绑定设备指纹/风控分。

## 5) 搜索端口专项防护

- 限制 `q` 参数长度和特殊字符比例。
- 对高成本查询加缓存（Redis）和最短查询间隔。
- 对匿名用户搜索做更低阈值，登录用户做分级配额。
- 分页上限（如 `size <= 50`），避免大页拖垮数据库。

## 6) 压测（合法授权）

使用 k6 验证限流阈值是否合理：

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    login_burst: {
      executor: 'constant-arrival-rate',
      rate: 30,
      timeUnit: '1s',
      duration: '2m',
      preAllocatedVUs: 50,
      exec: 'login',
    },
    search_burst: {
      executor: 'constant-arrival-rate',
      rate: 80,
      timeUnit: '1s',
      duration: '2m',
      preAllocatedVUs: 100,
      exec: 'search',
    },
  },
};

export function login() {
  http.post('https://your.domain/login', { username: 'u', password: 'p' });
  sleep(0.2);
}

export function search() {
  http.get('https://your.domain/search?q=java');
  sleep(0.1);
}
```

验收目标（示例）：

- 正常流量下：`P95 < 300ms`，`5xx < 0.5%`。
- 攻击流量下：网关出现大量 `429`，核心业务仍可用。
- CPU/连接池不过载，数据库慢查询不显著上升。

## 7) 监控与告警

最少要有这些指标：

- Nginx：`2xx/4xx/5xx`、`429` 比例、活跃连接数。
- Java：线程池队列长度、GC 暂停、Tomcat/Undertow 活跃线程。
- DB/Redis：连接数、慢查询、命中率。

告警建议：

- 1 分钟内 `429` 激增 + `5xx` 上升 => 疑似攻击。
- `/login` 失败率突增（例如 > 40%）=> 启动验证码强制策略。
- `/search` QPS 异常且重复关键词高 => 启用更严格限流。

## 8) 应急 5 分钟 Runbook

1. 网关下发更严限流（`/login` 与 `/search` 优先）。
2. 临时开启验证码 / 挑战页。
3. 封禁异常 IP 段 / ASN（注意误封评估）。
4. 将非核心接口降级（返回缓存或静态兜底）。
5. 观察 429、5xx、P95，逐步回调阈值。

---

如果你愿意，我可以下一步按你的实际栈（Spring Boot + Nginx/Apache）给出可直接落地的 `application.yml`、Filter/Interceptor 代码和 Nginx 完整配置文件。
