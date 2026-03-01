# 事故案例庫

> 每次解決新問題後，將案例補充至此。
> 格式：日期 / 症狀 / 根本原因 / 解法
> 讓 k8s-debugger 在遇到類似症狀時能直接參考。

---

## 案例模板

遇到新問題解決後，照以下格式補充：

```markdown
## [YYYY-MM-DD] <一句話描述問題>

**症狀**
- Pod 狀態：
- 錯誤訊息：

**根本原因**
說明為什麼會發生

**診斷關鍵線索**
哪個指令或哪段 log 讓你找到答案

**解法**
實際執行的修復步驟

**預防措施**
之後怎麼避免同樣問題
```

---

## 案例範例（示範格式）

## [2025-11-15] production api-service 因 ConfigMap 更新後無法啟動

**症狀**
- Pod 狀態：CrashLoopBackOff
- 錯誤訊息：`Error: config key 'DATABASE_POOL_SIZE' not found`

**根本原因**
部署新版本時更新了 ConfigMap，新增了必要的 key `DATABASE_POOL_SIZE`，
但 rolling update 時舊 Pod 先被刪除、新 ConfigMap 尚未套用，
導致新 Pod 啟動時找不到該 key。

**診斷關鍵線索**
`kubectl logs --previous` 直接顯示缺少的 config key 名稱

**解法**
```bash
# 確認 ConfigMap 是否有新 key
kubectl get configmap api-config -n production -o yaml | grep DATABASE_POOL_SIZE

# 若無，補上後重啟
kubectl set env deployment/api-service DATABASE_POOL_SIZE=10 -n production
```

**預防措施**
- ConfigMap 更新與 Deployment 更新應在同一次 apply
- 應用程式啟動時應明確驗證所有必要設定是否存在，而非執行到才報錯

---

## 新案例從這裡開始補充 ↓
