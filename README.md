# CI/CD Lab

> 學號：314551081  
> 作業 Workflow：[`.github/workflows/ci_314551081.yaml`](.github/workflows/ci_314551081.yaml)

## Homework: CI Pipeline (`ci_314551081.yaml`)

本作業在原始 lab 的基礎上，新增了一條 CI Pipeline，檔案位於 [`.github/workflows/ci_314551081.yaml`](.github/workflows/ci_314551081.yaml)。

### Pipeline 設計

| Stage           | 指令                           | 說明                                                             |
| --------------- | ------------------------------ | ---------------------------------------------------------------- |
| Checkout        | `actions/checkout@v4`          | 拉取本次 commit 原始碼                                           |
| Setup Node.js   | `actions/setup-node@v4`        | 在 Node 22 / 24 雙版本 matrix 下執行，並啟用 `cache: npm` 以加速 |
| Install         | `npm ci`                       | 以 lockfile 為基準乾淨安裝相依套件                               |
| **Typecheck**   | `npm run typecheck`            | TypeScript 靜態型別檢查（`tsc --noEmit`）                        |
| **Prettier**    | `npm run format:check`         | 全專案程式碼風格檢查（不通過會直接失敗）                         |
| **Test**        | `npm test -- --reporter=junit` | 以 vitest 執行單元測試並輸出 JUnit XML                           |
| Publish report  | `dorny/test-reporter@v1`       | 將 JUnit 結果直接呈現在 GitHub Actions Checks UI                 |
| Upload artifact | `actions/upload-artifact@v4`   | 保留 JUnit XML 供後續下載                                        |
| Job summary     | `$GITHUB_STEP_SUMMARY`         | 在 workflow run 頁產生 Markdown 摘要表                           |

### 觸發方式（push 自動執行）

```yaml
on:
  push:
    branches:
      - '**'
  pull_request:
  workflow_dispatch:
```

任何 branch 的 push 都會觸發；同時對 PR 與手動觸發開放。

### 失敗即失敗（不會誤判為 success）

每個檢查步驟都使用預設的 fail-fast 行為：

- `npm run typecheck` 一旦回傳非 0 exit code，該 step 會 fail，後續步驟不再執行成功訊號。
- `npm run format:check`、`npm test` 同上。
- `dorny/test-reporter@v1` 設定 `fail-on-error: true`，即使測試 step 後續又 publish report，整體 job 仍會被標記失敗。
- Matrix `fail-fast: false` 讓 Node 22 / 24 各自獨立判定，但任一版本失敗整體 workflow run 仍為紅燈。

### 測試結果在 GitHub Actions 顯示

使用 [`dorny/test-reporter@v1`](https://github.com/dorny/test-reporter)（Marketplace action）將 vitest 產出的 JUnit XML 解析成 GitHub Checks，會直接出現在 PR / commit / Actions run 頁面。`$GITHUB_STEP_SUMMARY` 額外提供一張各階段結果表格作為快速摘要。

### 進階用法（鼓勵分數的部分）

- **Matrix 多版本驗證**：同一份程式同時在 Node 22 / 24 跑，提早發現相容性問題。
- **`concurrency` 自動取消**：對同一 ref 的新 push 會取消上一次仍在執行的 run，省 CI 時間。
- **JUnit + Test Reporter**：將測試報告原生整合進 GitHub UI，不必下載 artifact 才看得到失敗測試。
- **Step Summary**：以 Markdown 表格列出各階段結果，commit metadata 一目了然。
- **Artifact 上傳**：JUnit XML 可下載做後續分析或長期保存。

### 在本機驗證（不必 push 也能跑）

```bash
npm ci
npm run typecheck
npm run format:check
npm test
```

或使用 `act` 模擬整條 workflow：

```bash
act push -W .github/workflows/ci_314551081.yaml
```

---

這份文件是 Lab 手冊，會帶你完成：

1. 啟動 Fastify 應用
2. 實際 push 到 GitHub 觀察 CI
3. 在本機用 `act` 模擬 CI
4. 理解不同 branch 推送時的效果
5. 完成課堂練習

## 開始前

建議使用 GitHub Codespaces 開啟本 repo（環境已預先準備 Node / Docker / act）。

### Fork

1. 到原始 repo 頁面，點右上角 **Fork**
2. 進入你自己的 fork repo
3. 用你的 fork repo 開啟 Codespaces

先確認工具版本：

```bash
node --version
docker --version
docker compose version
act --version
```

## 本地開發

安裝依賴：

```bash
npm ci
```

啟動服務：

```bash
npm run build
npm run start
```

驗證服務：

```bash
curl http://localhost:3000/
curl http://localhost:3000/health
```

## Lab-01: Hello GitHub Actions

Push 到 GitHub 觀察 CI

### 第一次使用請先確認 Actions 已啟用

在你的 fork repo：

1. 點選上方 **Actions** 分頁
2. 如果看到啟用提示，點 **I understand my workflows, go ahead and enable them**（或同義按鈕）
3. 回到 Code 頁面繼續操作

### 實際推送一個 feature branch

把 `snippets/01_hello.yaml` 複製到 `.github/workflows/` 底下

```bash
git checkout -b feature/ci-observe
cp snippets/01_hello.yaml .github/workflows/
git add .
git commit -m "ci: add hello.yaml"
git push origin feature/ci-observe
```

### 在 GitHub Actions 頁面觀察 Hello CI 結果

1. 到你的 fork repo 的 **Actions** 頁面
2. 找到最新 `ci` workflow run
3. 查看 01_hello.yaml 的執行步驟與結果

---

## Lab-02: Run test

把 `snippets/02_run-test.yaml` 複製到 `.github/workflows/` 底下

```bash
cp snippets/snippets/02_run-test.yaml .github/workflows/
git add .
git commit -m "ci: add run-test.yaml"
git push origin feature/ci-observe
```

### 在 GitHub Actions 頁面觀察 Run test CI 結果

- 觀察 run-test.yaml 內容
- 到 GitHub Actions 頁面, 查看 run test 執行結果
- 觀察 artifact

---

## Lab-03: Run GitHub Actions with act locally

### 用 `act` 模擬 push event

`act` 是一個可以在本機執行 GitHub Actions workflow 的工具

你可以用它：

- 在不 push 到 GitHub 的情況下先驗證 workflow
- 快速重跑失敗步驟、縮短除錯時間
- 模擬 `push` / `pull_request` 等事件

官方資源：

- 官網：<https://nektosact.com/>
- GitHub 專案：<https://github.com/nektos/act>

切到要模擬的 branch，執行 `act push`

```bash
act push
```

若只想跑單一 workflow，可加 `-W`：

```bash
act push -W .github/workflows/ci.yaml
```

進階補充：若你需要在「不切 branch」情況下指定事件分支，可以使用：

```bash
act push --env GITHUB_REF=refs/heads/<branch>
```

---

## Lab-04: Conditional workflow and deploying

實際應用中, 我們可能會將每一個版本都跑過 ci, 並 build 出 image
但 image tag 要可以區分出是 feature branch 或是 release branch
方便我們理解與追蹤特定版本的 source code

觀察 `snippets/ci.yaml` 與 `snippets/cd.yaml`

將 `snippets/ci.yaml` 與 `snippets/cd.yaml` 複製到 `.github/workflows/` 底下

```bash
cp snippets/ci.yaml .github/workflows/
cp snippets/cd.yaml .github/workflows/
```

### 在 feature branch 觀察 ci.yaml 執行結果

```bash
git checkout -b feature/a
act push
```

觀察 build 出來的 image

```bash
docker images
```

### 切出 release branch 觀察 ci.yaml 執行結果

切出 release branch

```bash
git checkout -b release/1.0.0
act push
```

觀察 build 出來的 image

```bash
docker images
```

## 思考

- 如何設計 CI Pipeline 以確保程式碼品質
- 如何設計 CD Pipeline 部署到目標環境
- CI Pipeline 與 CD Pipeline 的相依關係
- 通知或報表機制
