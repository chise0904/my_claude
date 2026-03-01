# CTBC 環境特有設定

> 這份文件記錄 CTBC 環境的特有慣例，避免 Claude 給出不適用的通用建議。
> 隨環境異動請持續更新。

## Namespace 命名慣例

```
production    → 正式環境
staging       → 預備環境
development   → 開發環境
monitoring    → Prometheus / Grafana / Loki
infra         → Kong、Traefik 等基礎設施
```

## 常用 Label 規範

所有 Deployment 應包含：
```yaml
labels:
  app: <service-name>
  version: <image-tag>
  env: <production|staging|development>
  team: <team-name>
```

## 資源 Requests/Limits 基準

| 服務類型 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------|------------|-----------|---------------|-------------|
| 輕量 API | 100m | 500m | 128Mi | 512Mi |
| 一般服務 | 250m | 1000m | 256Mi | 1Gi |
| 資料庫 sidecar | 50m | 200m | 64Mi | 256Mi |

*實際值依應用程式用量調整，上表為最低起始值*

## Image Registry

- 內部 registry：`registry.ctbc.internal`
- ImagePullSecret 名稱：`registry-secret`（各 namespace 都應有此 secret）

## 網路設定

- 所有外部流量透過 **Kong API Gateway** 進入
- 內部服務間通訊透過 **ClusterIP Service**
- 特殊情況需要 NodePort 請先與網路團隊確認

## 儲存設定

- StorageClass：`ceph-rbd`（預設）、`nfs-client`（共享檔案）
- PVC 申請需標注預計用量和保留時間

## 監控整合

**Prometheus Scrape Annotations（必須加）**
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

**Log 格式要求**
- 應用程式日誌必須輸出 JSON 格式
- 必要欄位：`timestamp`、`level`、`message`、`trace_id`
- Loki 透過 namespace + app label 自動收集

## 已知環境限制

- `kubectl exec` 在 production 需要額外申請權限
- `kubectl port-forward` 在 production 不開放
- 每個 namespace 有 ResourceQuota 限制，若部署失敗先確認 quota

## 憑證管理

- TLS 憑證由 cert-manager 自動管理
- Issuer 名稱：`letsencrypt-prod`（外部）、`internal-ca`（內部）
- 憑證快到期時 cert-manager 會自動 renew，但需確認 ACME challenge 能正常通過
