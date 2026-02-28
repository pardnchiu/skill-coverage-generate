> [!NOTE]
> 本文件提供英文版本：[README.md](./README.md)

# skill-coverage-generate [![license](https://img.shields.io/github/license/pardnchiu/skill-coverage-generate)](LICENSE)

> 一個 Claude Code Skill，自動分析 Go 測試覆蓋率（精確到每個函式），並生成 Table-Driven 測試案例，將專案覆蓋率提升至 ≥90% 門檻。
> 在任意 Go 專案目錄下呼叫 `/coverage-generate`，即可完成原始碼探索、覆蓋率量測、測試生成的完整端到端流程。

## 目錄

- [功能特點](#功能特點)
- [安裝](#安裝)
- [使用方法](#使用方法)
- [命令列參考](#命令列參考)
- [授權](#授權)

## 功能特點

- **自動化覆蓋率分析**：執行 `go test -coverprofile` 並解析各函式覆蓋率，識別低於 90% 的缺口
- **智能測試生成**：建立 Table-Driven 測試，涵蓋正常路徑、Nil 輸入、邊界條件、錯誤路徑與 Context 取消等場景
- **多套件支援**：透過 `./...` 處理整個 Go Module，一次執行涵蓋所有套件
- **智能排除機制**：自動跳過 `main.go`、生成檔案（`*.pb.go`、`*_generated.go`）及 `init()` 函式
- **品質門檻驗證**：以 `go build ./...` 確認編譯通過，並重新執行覆蓋率量測以驗證改善成效

## 安裝

```bash
git clone https://github.com/pardnchiu/skill-coverage-generate ~/.claude/skills/coverage-generate
```

## 使用方法

在 Go 專案目錄下呼叫技能：

```
/coverage-generate
```

Claude 將執行完整的四階段工作流程——探索、分析、生成、驗證——並回報改善前後的覆蓋率差異。

## 命令列參考

### 工作流程階段

| 階段 | 動作 | 工具 |
|------|------|------|
| 1. 探索 | 尋找非測試 Go 原始檔 | `find` |
| 2. 分析 | 生成覆蓋率報告 | `go test -coverprofile=coverage.out ./...` |
| 3. 生成 | 解析缺口並撰寫測試 | `go tool cover -func=coverage.out` |
| 4. 驗證 | 編譯並重新量測覆蓋率 | `go build ./...`、`go test ./...` |

### 排除檔案

| 模式 | 原因 |
|------|------|
| `*_test.go` | 已是測試檔案 |
| `*.pb.go`、`*_generated.go` | 自動生成的程式碼 |
| `main.go` | 屬於整合測試範疇 |
| `init()` 函式 | 無法直接測試 |

## 授權

本專案採用 [MIT LICENSE](LICENSE)。
