# GitHub 開發流程練習筆記

用 Claude Code 從零走過一次完整的 GitHub 協作流程:開發 → commit → push →
GitHub Actions CI → Pull Request → branch protection → commit message 規範。

本筆記按實際練習順序記錄每一步、對應指令,以及過程中踩到的坑與學到的觀念。

---

## 0. 前置環境

| 工具 | 用途 | 確認指令 |
| --- | --- | --- |
| git | 版本控制 | `git --version` |
| GitHub CLI (`gh`) | 建 repo / 開 PR / 呼叫 API | `gh --version`、`gh auth status` |
| Node.js + npm | 專案執行與測試 | `node --version`、`npm --version` |

> `gh` 不是必要的 —— 建 repo、push、開 PR 都能用純 git + 網頁完成。
> 它只是把這些步驟濃縮成指令,方便自動化。

---

## 1. 建立本地專案

一個最小的 TypeScript 專案(math 函式 + vitest 測試),用來當練習載體。

```
package.json      # 定義 build / test / dev 指令與 devDependencies
tsconfig.json     # TypeScript 編譯設定
src/math.ts       # add / multiply 等函式
src/math.test.ts  # vitest 測試
src/index.ts      # 進入點
.gitignore        # 忽略 node_modules / dist
README.md
```

```bash
npm install
npm run build   # tsc 編譯,抓型別錯誤
npm test        # vitest 跑測試
```

---

## 2. 初始化 git 並建立第一個 commit

```bash
git init
git add .
git commit -m "Initial commit: ..."
git branch -M main   # 把預設分支改名為 main
```

---

## 3. 用 gh 建立遠端 repo 並 push

```bash
gh repo create claude-code-practice --private --source=. --remote=origin --push
```

一次完成:在 GitHub 建立 repo、設定 `origin` remote、把 `main` push 上去。

---

## 4. 加入 GitHub Actions CI

`.github/workflows/ci.yml`:每次 push 到 main 或對 main 開 PR 時,自動在雲端跑
checkout → 裝 Node → `npm ci` → `npm run build` → `npm test`。

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - run: npm test
```

**CI 的意義**:任何一步失敗,整個 workflow 標記為失敗,commit / PR 旁會出現紅叉,
自動把關「這次改動可能有問題」。

**怎麼查 CI 結果**：

```bash
gh run list --limit 5        # 列出最近幾次執行
gh run view --log            # 看某次的完整 log
gh pr checks <PR編號>         # 看某個 PR 上的 check 狀態
```

---

## 5. Feature 分支 → PR → merge 的標準循環

```bash
git checkout -b feature/xxx   # 開分支
# ...改程式碼、加測試...
npm test                      # 本地先驗證
git add . && git commit -m "..."
git push -u origin feature/xxx

gh pr create --base main --head feature/xxx --title "..." --body "..."
gh pr checks <PR編號>          # 等 CI 綠燈
# 在網頁按 Squash and merge(或 gh pr merge)
```

**Merge 後同步本地**：

```bash
git checkout main
git pull                       # 抓下 squash 後的新 commit
git branch -d feature/xxx      # 刪本地已合併分支
git fetch --prune              # 清掉遠端已刪分支的追蹤參考
```

### 三種合併方式

| 方式 | 效果 |
| --- | --- |
| Create a merge commit | 保留完整歷史,額外產生一個 merge commit |
| **Squash and merge** | 把分支所有 commit 壓成一個新 commit 接到 main（main 歷史乾淨）|
| Rebase and merge | commit 逐一接到 main 後面,不產生 merge commit |

> 本練習全程用 squash merge,所以合併後的 commit hash 會跟分支上的不同(是重寫的新 commit)。

---

## 6. 練習：故意弄壞 CI

把測試斷言改成錯的(`expect(add(2,3)).toBe(999)`)→ push → 開 PR，
觀察 CI 從綠變紅，證明 CI 真的在檢查、不是裝飾。再修回去 → CI 變綠 → 合併。

這就是實務最常見的「PR 被 CI 擋下 → 修正 → 重新通過」循環。

---

## 7. Branch Protection（分支保護）—— 踩坑重點

目標：要求「CI 通過 + 走 PR」才能進 main。過程踩到多個坑：

| 情境 | 結果 | 學到的事 |
| --- | --- | --- |
| **Private repo + 免費個人帳號** | 規則能存檔，但**完全不生效** | 網頁會顯示警告「rules won't be enforced ... until you move to a GitHub Team or Enterprise organization account」。設定介面正常不代表有作用，**一定要實測** |
| 直接 push 到 main（規則未生效） | push 成功 | 驗證了上面的警告是真的 |
| **改成 public repo** | 規則開始生效 | 但 push 仍成功，訊息出現 `Bypassed rule violations` |
| Public + admin 未禁止繞過 | admin 可繞過規則 | 預設允許 admin bypass，GitHub 會誠實標示「這次是被允許繞過的」 |
| 勾選 **Do not allow bypassing** 後 push | **被擋下** `GH006: Protected branch ... Changes must be made through a pull request` | 這才是「連 admin 都擋住」的完整保護 |
| Require approvals = 1（一人專案） | PR 永遠無法合併 | **GitHub 不允許自己核准自己的 PR**，一人專案要把核准數設為 0 |

**結論**：免費方案要真正用到 branch protection，需把 repo 改成 **public**，
或建立 **organization**（組織帳號）。個人帳號的 private repo 規則不生效。

**用 API 查/設定規則**（repo 為 public 時）：

```bash
gh api repos/<owner>/<repo>/branches/main/protection
gh api repos/<owner>/<repo>/branches/main/protection/required_status_checks
```

> 陷阱：畫面上勾了「Require status checks」但沒把具體 check 加進清單時，
> `checks` 其實是空的 `[]`，等於沒有強制任何檢查。要在搜尋框把 check 名稱加進去。

---

## 8. 讓 main 的每個 commit message 都帶「單號」

需求：模擬外部系統工作單號（類 Jira，如 `PROJ-123`），
確保 main 上每筆 commit message 都有單號。

**原理**：squash merge 的 commit message 預設 = PR 標題，
所以只要「PR 標題有單號」且「用 CI 強制驗證」，落到 main 的 commit 自然帶單號。

### 做法

1. **CI 加一個驗證 job**，用正規表示式檢查 PR 標題開頭是 `[PROJ-123]` 格式：

```yaml
  validate-pr-title:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Check PR title starts with a ticket number
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
        run: |
          if [[ ! "$PR_TITLE" =~ ^\[[A-Z]+-[0-9]+\] ]]; then
            echo "PR title must start with a ticket number, e.g. [PROJ-123] ..."
            exit 1
          fi
```

2. **把 `validate-pr-title` 加進 branch protection 的必要檢查清單**
   （連同 `build-and-test`），沒過就無法合併。

3. **讓「編輯 PR 標題」也重跑驗證**——預設 `pull_request` 只在
   opened / synchronize / reopened 觸發，不含編輯標題。加上 `edited`：

```yaml
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, edited]
```

### 驗證結果

- 標題有單號 → 通過 → 合併後 main 的 commit message 自動帶單號 ✅
- 標題沒單號 → `validate-pr-title` 失敗 → 合併被擋 ✅
- 改標題 → 自動重跑驗證（`edited` 生效）✅
- 標題開頭多一個空格 → 也被擋（正規表示式 `^\[` 要求開頭就是 `[`）

### 機制的邊界（重要）

- **Squash merge 確認框裡的「Commit message」可以臨時手改，CI 擋不住這一步。**
  格式驗證擋得住「PR 標題」，擋不住「合併當下的手動竄改」。
  要補洞需另加 `on: push` 檢查落地 commit（但那是事後偵測）。
- **CI 只能驗證「格式」，不能驗證「語意」。** commit 描述寫得貼不貼切,
  仍要靠人（例：曾複製到不相符的舊標題，格式過了但內容不符）。

---

## 常用指令速查

```bash
# Repo / PR
gh repo create <name> --private --source=. --remote=origin --push
gh pr create --base main --head <branch> --title "..." --body "..."
gh pr checks <PR編號>
gh pr view <PR編號> --json mergeable,mergeStateStatus
gh pr merge <PR編號>

# CI
gh run list --limit 5
gh run view --log

# 分支清理
git branch -d <branch>                 # 刪本地已合併分支
git fetch --prune                      # 清遠端已刪分支的追蹤
gh api -X DELETE repos/<owner>/<repo>/git/refs/heads/<branch>   # 刪遠端分支

# Branch protection（需 public 或組織帳號）
gh api repos/<owner>/<repo>/branches/main/protection
```

---

## 一句話總結

CI 能自動把關「格式與正確性的可驗證部分」（build/test/標題格式），
branch protection 能強制「流程必須被遵守」（走 PR、check 要過、連 admin 都不例外）——
但兩者都有邊界，語意正確與合併當下的手動操作，終究仍需要人的紀律。
