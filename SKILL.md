# Go 測試覆蓋率生成器技能

## 目的
分析 Go 專案的測試覆蓋率,並自動生成/更新測試檔案以達到 ≥90% 的覆蓋率門檻。

## 工作流程概覽

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  1. 探索        │ ──▶ │  2. 覆蓋率       │ ──▶ │  3. 生成        │
│  列出 .go 檔案  │     │  分析            │     │  缺失的測試     │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

---

## 階段 1:探索

### 指令序列
```bash
# 導航至 Go 模組根目錄(go.mod 所在位置)
cd /path/to/module

# 列出所有 Go 原始檔案(排除測試、vendor、生成的檔案)
find . -name "*.go" \
  ! -name "*_test.go" \
  ! -path "./vendor/*" \
  ! -path "./.git/*" \
  ! -name "*.pb.go" \
  ! -name "*_generated.go" \
  -type f
```

### 輸出格式
儲存探索到的檔案供後續階段使用:
```
./cmd/main.go
./internal/service/user.go
./internal/repository/db.go
./pkg/utils/helper.go
```

---

## 階段 2:覆蓋率分析

### 步驟 2.1:生成覆蓋率分析檔
```bash
# 執行測試並產生覆蓋率(包含所有套件)
go test -coverprofile=coverage.out -covermode=atomic ./...

# 如果特定套件失敗,繼續執行其他套件
go test -coverprofile=coverage.out -covermode=atomic ./... 2>&1 || true
```

### 步驟 2.2:解析覆蓋率報告
```bash
# 取得各函式的覆蓋率明細
go tool cover -func=coverage.out
```

### 預期輸出格式
```
github.com/user/repo/internal/service/user.go:25:    CreateUser      85.7%
github.com/user/repo/internal/service/user.go:48:    GetUser         100.0%
github.com/user/repo/internal/service/user.go:62:    DeleteUser      0.0%
github.com/user/repo/pkg/utils/helper.go:12:         ParseID         66.7%
total:                                               (statements)    72.3%
```

### 步驟 2.3:識別低覆蓋率檔案
解析輸出並篩選覆蓋率 < 90% 的檔案:

```bash
# 提取檔案層級的覆蓋率(彙總每個檔案的函式)
go tool cover -func=coverage.out | \
  grep -v "total:" | \
  awk -F: '{files[$1]+=$NF; count[$1]++} END {for(f in files) print f, files[f]/count[f]}' | \
  awk '$2 < 90 {print $1, $2"%"}'
```

### 覆蓋率資料結構
對於每個低於門檻的檔案,收集:
```
{
  "file": "./internal/service/user.go",
  "current_coverage": 62.3,
  "uncovered_functions": [
    {"name": "DeleteUser", "line": 62, "coverage": 0.0},
    {"name": "UpdateUser", "line": 85, "coverage": 45.2}
  ]
}
```

---

## 階段 3:測試生成

### 步驟 3.1:分析原始檔案
對於每個低覆蓋率檔案,提取:

```bash
# 解析匯出的函式及其簽章
go doc -all ./internal/service
```

或使用 AST 檢查:
```bash
# 列出所有函式及其簽章
grep -n "^func " ./internal/service/user.go
```

### 步驟 3.2:檢查現有測試檔案
```bash
# 確定測試檔案路徑
SOURCE_FILE="./internal/service/user.go"
TEST_FILE="${SOURCE_FILE%.go}_test.go"

# 檢查是否存在
if [ -f "$TEST_FILE" ]; then
  # 提取現有的測試函式名稱
  grep "^func Test" "$TEST_FILE" | sed 's/func \(Test[^(]*\).*/\1/'
fi
```

### 步驟 3.3:生成測試程式碼

#### 測試檔案範本
```go
package <package_name>

import (
    "testing"
    // 根據函式簽章新增必要的匯入
)

// TestFunctionName 測試 FunctionName 函式。
// 覆蓋率目標:錯誤路徑、邊界情況、正常流程。
func TestFunctionName(t *testing.T) {
    tests := []struct {
        name    string
        // 輸入欄位
        want    // 預期輸出類型
        wantErr bool
    }{
        {
            name:    "nominal case",
            // 輸入
            want:    // 預期
            wantErr: false,
        },
        {
            name:    "nil input",
            // nil/空輸入
            want:    // 零值或特定值
            wantErr: true,
        },
        {
            name:    "boundary condition",
            // 邊界情況輸入
            want:    // 預期
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := FunctionName(/* tt.inputs */)
            if (err != nil) != tt.wantErr {
                t.Errorf("FunctionName() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("FunctionName() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

#### 生成規則

1. **表格驅動測試**:始終使用表格驅動模式以利維護
2. **測試案例必須包含**:
   - 正常/成功路徑
   - Nil/空輸入處理
   - 錯誤條件(如果函式回傳錯誤)
   - 邊界值(針對數值輸入)
   - Context 取消(如果 context.Context 是參數)

3. **命名慣例**: `Test<FunctionName>_<Scenario>`
   ```go
   func TestCreateUser(t *testing.T) { ... }
   func TestCreateUser_NilInput(t *testing.T) { ... }
   func TestCreateUser_DuplicateEmail(t *testing.T) { ... }
   ```

4. **方法接收者**:對於結構方法,正確實例化
   ```go
   func TestService_CreateUser(t *testing.T) {
       svc := &Service{
           repo: mockRepo,
           log:  mockLogger,
       }
       // 測試邏輯
   }
   ```

5. **介面模擬**:為依賴項生成模擬實作
   ```go
   type mockRepository struct {
       createFunc func(ctx context.Context, u *User) error
   }

   func (m *mockRepository) Create(ctx context.Context, u *User) error {
       if m.createFunc != nil {
           return m.createFunc(ctx, u)
       }
       return nil
   }
   ```

### 步驟 3.4:寫入或更新測試檔案

#### 新檔案
```bash
# 建立新測試檔案
cat > "${TEST_FILE}" << 'EOF'
// Generated test file - review and customize test cases
<generated_content>
EOF
```

#### 現有檔案(追加)
```bash
# 將新測試追加到現有檔案(在最後的結束大括號之前,如果有的話)
# 或直接追加到結尾
cat >> "${TEST_FILE}" << 'EOF'

// --- Auto-generated tests below ---
<new_test_functions>
EOF
```

---

## 驗證

### 生成後驗證
```bash
# 1. 確保測試可編譯
go build ./...

# 2. 執行新測試
go test -v ./...

# 3. 重新生成覆蓋率報告
go test -coverprofile=coverage_new.out ./...
go tool cover -func=coverage_new.out | grep "total:"

# 4. 比較改進情況
echo "Previous: $(grep total coverage.out)"
echo "Current:  $(grep total coverage_new.out)"
```

### 品質門檻
- [ ] 所有生成的測試都能正確編譯
- [ ] 沒有引入匯入循環
- [ ] 測試命名遵循慣例
- [ ] 目標檔案的覆蓋率已改善

---

## 使用範例

### 完整工作流程執行
```bash
# 1. 探索
echo "=== Phase 1: Discovery ==="
GO_FILES=$(find . -name "*.go" ! -name "*_test.go" ! -path "./vendor/*" -type f)
echo "$GO_FILES"

# 2. 覆蓋率分析
echo "=== Phase 2: Coverage Analysis ==="
go test -coverprofile=coverage.out ./... 2>&1 || true
go tool cover -func=coverage.out

# 3. 識別目標(< 90%)
echo "=== Files below 90% coverage ==="
LOW_COVERAGE=$(go tool cover -func=coverage.out | grep -v total | awk '{if($3+0 < 90) print $1}' | sort -u)
echo "$LOW_COVERAGE"

# 4. 對每個低覆蓋率檔案生成測試
for file in $LOW_COVERAGE; do
    echo "Generating tests for: $file"
    # Claude 分析檔案並生成適當的測試
done
```

---

## 邊界情況與考量事項

### 需要跳過的檔案
- `package main` 中的 `main.go`(整合測試範疇)
- 生成的檔案(`*.pb.go`、`*_generated.go`)
- 僅包含類型定義的檔案(沒有函式)

### 複雜場景
1. **未匯出的函式**:在相同套件中生成測試(不是 `_test` 套件)
2. **init() 函式**:無法直接測試,跳過
3. **CGO 依賴**:可能需要在測試檔案中使用建置標籤
4. **外部服務呼叫**:必須模擬;透過介面參數識別

### 錯誤處理
- 如果 `go test` 完全失敗:檢查缺少的依賴項(`go mod tidy`)
- 如果覆蓋率為 0%:驗證測試檔案是否在正確的套件中
- 如果函式顯示 0% 但測試存在:檢查測試是否實際呼叫了該函式

---

## 檔案結構參考

```
project/
├── go.mod
├── coverage.out          # 生成的覆蓋率分析檔
├── cmd/
│   └── main.go          # 單元測試跳過
├── internal/
│   └── service/
│       ├── user.go      # 原始檔案
│       └── user_test.go # 測試檔案(生成/更新)
└── pkg/
    └── utils/
        ├── helper.go
        └── helper_test.go
```