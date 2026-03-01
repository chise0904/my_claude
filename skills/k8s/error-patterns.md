# K8s 常見錯誤模式

## CrashLoopBackOff

**症狀**：Pod 不斷重啟，Restart Count 持續增加

**診斷優先順序**
1. `kubectl logs <pod> --previous` — 看崩潰前最後的錯誤訊息
2. `kubectl describe pod` 的 `Last State.Exit Code`

**Exit Code 對照表**
| Exit Code | 意義 | 常見原因 |
|-----------|------|---------|
| 1 | 應用程式錯誤 | 設定檔錯誤、DB 連線失敗、缺少環境變數 |
| 137 | OOMKilled (SIGKILL) | memory limit 太低 |
| 139 | Segfault | 應用程式 bug |
| 143 | SIGTERM | liveness probe 失敗觸發重啟 |

**Liveness Probe 過嚴導致的假 CrashLoop**
- 症狀：應用程式啟動慢，但 probe 太快開始檢查
- 解法：調整 `initialDelaySeconds`（建議比實際啟動時間多 20 秒）
- 或改用 `startupProbe` 讓啟動期間有更長容忍時間

---

## OOMKilled

**症狀**：Exit Code 137，describe 顯示 `OOMKilled: true`

**診斷步驟**
```bash
# 確認目前 limit 設定
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}'

# 確認實際用量趨勢（需 metrics-server）
kubectl top pod <pod> --containers
```

**常見原因**
- Memory limit 設定過低（初始設定未根據實際用量調整）
- 記憶體洩漏（Memory Leak）— 用量隨時間持續增長
- 突發流量導致瞬間記憶體暴增

**處理方式**
- 短期：調高 memory limit（建議設為 P99 用量的 1.5 倍）
- 長期：調查是否有 memory leak，檢查 JVM heap 設定（Java 應用）

---

## Pending

**症狀**：Pod 一直停在 Pending，沒有被排程到節點

**診斷**：`kubectl describe pod` 的 Events 區段是關鍵

**Events 訊息對照**
| Events 訊息 | 原因 | 解法 |
|------------|------|------|
| `Insufficient cpu` | 節點 CPU 不足 | 調低 requests 或擴充節點 |
| `Insufficient memory` | 節點 Memory 不足 | 同上 |
| `no nodes available to schedule pods` | 所有節點都無法排程 | 確認 taint/toleration 設定 |
| `didn't match node selector` | nodeSelector 不符 | 確認節點 label |
| `didn't match Pod's node affinity` | affinity 規則過嚴 | 放寬 affinity 條件 |
| `pod has unbound PVC` | PVC 未被綁定 | 確認 StorageClass 和 PV 狀態 |

---

## ImagePullBackOff / ErrImagePull

**症狀**：Container 無法拉取 image

**診斷步驟**
```bash
kubectl describe pod <pod> | grep -A5 "Events"
# 看 image 名稱是否正確
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'
```

**常見原因與解法**
| 原因 | 解法 |
|------|------|
| Image 名稱或 tag 錯誤 | 確認 registry 上是否存在 |
| Private registry 無認證 | 確認 imagePullSecret 存在且綁定到 ServiceAccount |
| Registry 連線問題 | 確認節點能否連到 registry（proxy 設定） |
| Image 被刪除 | 重新 build 並 push |

---

## Terminating 卡住

**症狀**：`kubectl delete pod` 後 Pod 一直停在 Terminating

**診斷**
```bash
kubectl get pod <pod> -o json | jq '.metadata.finalizers'
kubectl get pod <pod> -o json | jq '.metadata.deletionTimestamp'
```

**原因**
- Finalizer 未被清除（通常是 controller 掛了）
- 應用程式沒有處理 SIGTERM，等到 `terminationGracePeriodSeconds` 才被強殺

**解法**
```bash
# 強制移除 finalizer（謹慎使用）
kubectl patch pod <pod> -p '{"metadata":{"finalizers":[]}}' --type=merge

# 強制刪除（最後手段）
kubectl delete pod <pod> --force --grace-period=0
```

---

## 節點相關

**NotReady 節點診斷**
```bash
kubectl describe node <node-name>
# 看 Conditions 區段的 Ready、MemoryPressure、DiskPressure、PIDPressure
```

**常見節點問題**
| Condition | 原因 |
|-----------|------|
| MemoryPressure=True | 節點記憶體不足，kubelet 開始驅逐 Pod |
| DiskPressure=True | 磁碟空間不足，通常是 /var/log 或 container image 佔滿 |
| PIDPressure=True | Process 數量達到上限 |
