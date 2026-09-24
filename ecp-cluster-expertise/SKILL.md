---
name: ecp-cluster-expertise
description: Chainsea ECP 叢集架構：Redis Session 共享、多節點 Tomcat、負載均衡前置條件
triggers:
  - 叢集
  - cluster
  - 多節點
  - copy 一份同時跑
  - 水平擴展
  - session 共享
  - RedissonSessionManager
  - aipower.xml
---

# ECP Cluster Architecture Expertise

## The Insight

ECP 的叢集能力已內建但**預設關閉**。Redisson Session Manager 的啟動條件是
`conf/Catalina/localhost/aipower.xml` 這個檔案存在。光把專案 copy 一份同機跑
**不等於叢集**，因為 port 會衝突、DB 和 Redis 各自獨立。

## Why This Matters

使用者直覺上認為「copy → 雙開 = 叢集」，但實際上叢集有三個**必要條件**：
1. **共用同一個 Redis**（Session 才能在節點間共享）
2. **共用同一個 MariaDB**（資料才一致）
3. **有負載均衡器**在前面分流（Nginx / HAProxy）

缺任何一個都不是真正的叢集，只是兩個獨立的單機系統。

## Recognition Pattern

- 使用者說「copy 一份同時跑」
- 使用者問「怎麼做水平擴展」
- 使用者問「session 會不會遺失」（切換節點後登出問題）
- Tomcat log 出現 `RedisConnectionException`（Redis 沒起或設定錯）

## 架構圖

```
            用戶請求
               │
      [Nginx 負載均衡 :80/:443]
         ┌─────┴─────┐
    [Tomcat A]   [Tomcat B]
    :22821        :22823
         │             │
         └──────┬──────┘
           [Redis :6379]      ← 所有節點共用
           [MariaDB :3306]    ← 所有節點共用
```

## 同機雙節點設定差異

| 設定項目 | 節點 A（主） | 節點 B（副） |
|---------|------------|------------|
| Tomcat HTTP port (`server.xml`) | 22821 | 22823 |
| Tomcat HTTPS port (`server.xml`) | 22822 | 22824 |
| Redis | 自己啟動 | **不啟動**，`redisson.yaml` 指向 A 的 127.0.0.1:6379 |
| MariaDB | 自己啟動 | **不啟動**，DB 連線設定指向 A |
| `aipower.xml` | 存在（已啟用） | 存在（已啟用） |
| `server.bat` | 完整啟動 | 只啟動 Tomcat，不啟動 Redis/MariaDB |

## 不同機器設定差異

B 機器的 `redisson.yaml`：
```yaml
singleServerConfig:
  address: "redis://[A機器IP]:6379"
  database: 10
```

B 機器的 DB 連線設定改指向 A 機器 IP（在 `tool/config/` 的 datasource 設定）。

## Redisson Session Manager 啟用條件

`aipower.xml`（`conf/Catalina/localhost/aipower.xml`）：
```xml
<?xml version="1.0" encoding="utf-8"?>
<Context>
    <Manager className="org.redisson.tomcat.RedissonSessionManager"
        configPath="${catalina.base}/conf/redisson.yaml"
        readMode="MEMORY"
        updateMode="DEFAULT"
    />
</Context>
```

- **readMode=MEMORY**：Session 資料載入後快取在 JVM 記憶體，讀取不走 Redis（效能佳）
- **updateMode=DEFAULT**：只有 session 被修改時才寫回 Redis
- 刪除此檔案 → 回到 Tomcat 預設 In-Memory Session（單機模式）

## 已內建的 Redis 相關檔案

| 檔案 | 說明 |
|------|------|
| `chainsea/redis/redis-server.exe` | 5.0.14.1 portable binary |
| `chainsea/redis/redis.conf` | 綁定 127.0.0.1、256MB、LRU、db10 |
| `chainsea/apache-tomcat/lib/redisson-all-3.37.0.jar` | Redisson 客戶端 |
| `chainsea/apache-tomcat/lib/redisson-tomcat-9-3.37.0.jar` | Tomcat Session 整合 |
| `chainsea/apache-tomcat/conf/redisson.yaml` | Redis 連線設定 |
| `chainsea/apache-tomcat/conf/Catalina/localhost/context-example.txt` | 原始範例（已實作為 aipower.xml） |

## 驗證叢集 Session 共享

```powershell
# 確認 Redis db10 有 session key
C:\com\chainsea\redis\redis-cli.exe -n 10 dbsize

# 列出所有 session keys
C:\com\chainsea\redis\redis-cli.exe -n 10 keys "*"
```

登入後 dbsize 應 > 0；在 A 節點登入後切到 B 節點，session 應仍有效。

---

## Conformance Addendum

## When to Use
Chainsea ECP 叢集架構：Redis Session 共享、多節點 Tomcat、負載均衡前置條件

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.
