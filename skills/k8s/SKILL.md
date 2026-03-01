# K8s 診斷知識庫

這個 skill 包含 CTBC 環境的 Kubernetes 診斷知識，在診斷 K8s 問題時使用。

## 知識文件清單

| 檔案 | 內容 |
|------|------|
| `error-patterns.md` | 常見錯誤模式與對應診斷方向 |
| `ctbc-env.md` | CTBC 環境特有設定與慣例 |
| `postmortem-cases.md` | 過去事故案例（持續補充） |

## 使用方式

診斷問題時，先讀 `error-patterns.md` 對照症狀，
若問題與環境設定有關則讀 `ctbc-env.md`，
若需要參考過去案例則讀 `postmortem-cases.md`。

## 補充知識

遇到新問題解決後，將案例補充到 `postmortem-cases.md`，
若發現新的 error pattern 則補充到 `error-patterns.md`。
