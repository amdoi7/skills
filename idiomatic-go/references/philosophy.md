# 设计哲学与危险信号 <!-- design-philosophy-and-red-flags -->

> This reference is loaded as an **escalation path** from SKILL.md.
> Prerequisite: you have already confirmed the problem is structural, not a local type/error/test issue.

## Table of Contents

- [数据验证前移 (Parse, Don't Validate)](#数据验证前移-parse-dont-validate)
- [Linus Torvalds: 好品味 (Good Taste)](#linus-torvalds-好品味-good-taste)
- [John Ousterhout: 复杂性管理 (Complexity Management)](#john-ousterhout-复杂性管理-complexity-management)
- [Google Code Review: 代码健康 (Code Health)](#google-code-review-代码健康-code-health)
- [危险信号清单 (Red Flags)](#危险信号清单-red-flags)
- [经典语录 (Quotes)](#经典语录-quotes)

---

## 数据验证前移 (Parse, Don't Validate)

> "Parse, don't validate." — Alexis King
>
> "Make illegal states unrepresentable." — Yaron Minsky

### 核心理念

Go 是基于数据结构的语言。**好的数据结构让错误状态无法表示**，从而在编译期或数据入口处消除错误，而非在运行时反复检查。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         系统边界                                    │
│  ┌───────────┐    ┌───────────────┐    ┌─────────────────────────┐  │
│  │ 原始输入  │ →  │ 解析 & 验证   │ →  │ 强类型领域对象 (可信)   │  │
│  │ (不可信)  │    │ (边界层)      │    │ 内部代码无需再验证      │  │
│  └───────────┘    └───────────────┘    └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 原则

| 原则                        | 说明                    |
| ------------------------- | --------------------- |
| **边界验证一次**                | 在系统入口处完成所有验证，内部代码信任上游 |
| **解析而非验证**                | 不要验证后丢弃信息，而是解析成强类型    |
| **类型即文档**                 | 用类型系统表达约束，而非注释或运行时检查  |
| **信任内部数据**                | 一旦数据通过边界，内部函数不应重复验证   |
| **Fail fast at boundary** | 边界层快速失败，暴露上游问题        |

### 与领域建模配合

- **先修局部数据模型，再决定是否需要改结构**：优先消除 stringly typed 参数、重复验证、error 语义混乱；不要一上来重画包树。
- **名称先在当前作用域表达职责**：先让包内类型名、方法名、变量名准确表达责任；不要默认扩展成整仓库命名整理。
- **允许领域依赖，隔离技术依赖**：订单依赖客户额度是正常业务事实；DB 表结构、HTTP DTO、框架对象才应该被隔离在边界。
- **不变式内建，策略外置**：对象成立所必需的约束放进类型和构造；可变运营规则交给显式策略或应用层编排。
- **只有当结构性症状持续存在时才升级**：例如重复 DTO churn、pass-through 层、跨包 ownership 混乱、测试 seam 很不自然。

### Go 实践示例

```go
// ❌ 到处验证 - 不信任数据
func processOrder(orderID string, amount float64) error {
    if orderID == "" {
        return errors.New("empty order ID")
    }
    if amount <= 0 {
        return errors.New("invalid amount")
    }
    // ... 后续每个函数都重复这些检查
}

func saveOrder(orderID string, amount float64) error {
    if orderID == "" {  // 又验证一次
        return errors.New("empty order ID")
    }
    // ...
}
```

```go
// ✅ 边界验证 + 强类型 - 解析而非验证
type OrderID string  // 非空，已验证

func ParseOrderID(s string) (OrderID, error) {
    if s == "" {
        return "", errors.New("empty order ID")
    }
    if !isValidOrderFormat(s) {
        return "", errors.New("invalid order ID format")
    }
    return OrderID(s), nil  // 解析成功，后续可信任
}

type Money struct {  // 金额永远 > 0，已验证
    cents int64  // 用分存储，避免浮点问题
}

func ParseMoney(amount float64) (Money, error) {
    if amount <= 0 {
        return Money{}, errors.New("amount must be positive")
    }
    return Money{cents: int64(amount * 100)}, nil
}

// 内部函数接收强类型，无需验证
func processOrder(id OrderID, amount Money) error {
    // id 一定非空，amount 一定 > 0
    // 直接处理业务逻辑，无需防御性检查
    return saveOrder(id, amount)
}

func saveOrder(id OrderID, amount Money) error {
    // 信任上游，直接操作
    return db.Insert(id, amount)
}
```

### 边界层设计

```go
// API Handler = 边界层，负责解析和验证
func CreateOrderHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 解析原始输入
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // 2. 验证并转换为领域对象
    orderID, err := ParseOrderID(req.OrderID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    amount, err := ParseMoney(req.Amount)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // 3. 调用内部服务，传递强类型（内部完全信任）
    if err := orderService.Process(r.Context(), orderID, amount); err != nil {
        http.Error(w, "processing failed", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusCreated)
}
```

### 类型设计模式

```go
// 1. NewType 模式 - 编译期区分
type UserID int64
type OrderID int64

func GetUser(id UserID) *User     // 不能传 OrderID
func GetOrder(id OrderID) *Order  // 不能传 UserID

// 2. 构造函数封装验证
type Email struct {
    value string  // 小写，私有
}

func ParseEmail(s string) (Email, error) {
    s = strings.ToLower(strings.TrimSpace(s))
    if !emailRegex.MatchString(s) {
        return Email{}, errors.New("invalid email")
    }
    return Email{value: s}, nil
}

func (e Email) String() string { return e.value }

// 3. 枚举限制取值
type OrderStatus int

const (
    OrderStatusPending OrderStatus = iota + 1
    OrderStatusPaid
    OrderStatusShipped
)

func ParseOrderStatus(s string) (OrderStatus, error) {
    switch s {
    case "pending":
        return OrderStatusPending, nil
    case "paid":
        return OrderStatusPaid, nil
    case "shipped":
        return OrderStatusShipped, nil
    default:
        return 0, fmt.Errorf("invalid status: %s", s)
    }
}
```

### 反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|----------|
| 到处 `if x == nil` | 不信任数据，代码臃肿 | 边界层确保非空，内部信任 |
| `string` 表示 ID | 类型不区分，易混淆 | NewType: `type UserID string` |
| `float64` 表示金额 | 精度丢失，无约束 | `Money{cents int64}` |
| 机械地让内部辅助函数返回 error | 把已被边界/类型消除的失败继续暴露给调用方 | 对可恢复失败返回 error；对已保证成立的不变式直接假设有效；仅对程序员 bug 或真正不可能状态 panic |
| 验证后仍用原始类型 | 验证信息丢失 | 解析成强类型 |
| `handler/service/repo` 成为主包结构 | 技术结构淹没业务概念 | 只有当这真的是复杂度来源时，才按业务能力重组；否则先在现有包内消除重复与 pass-through |

---

## Linus Torvalds: 好品味 (Good Taste)

> "Sometimes you can see a problem in a different way and rewrite it so that the special case goes away and becomes the normal case."

### 核心原则

| 原则                        | 说明                                 |
| ------------------------- | ---------------------------------- |
| **消除特殊情况**                | 边界情况应该通过设计消除，而不是通过条件判断处理           |
| **数据结构优先**                | 好程序员担心数据结构，糟糕程序员担心代码               |
| **嵌套限制**                  | 超过 3 层嵌套说明代码需要重构                   |
| **函数短小**                  | 函数应该短小精悍，只做一件事                     |
| **局部变量限制**                | 局部变量不应超过 5-10 个，否则需要拆分函数           |
| **Never break userspace** | 用户可见行为不变是神圣不可侵犯的铁律                 |
| **快速暴露问题**                | 不要写 fallback/兼容/回退代码，让上游数据问题在测试中暴露 |

### Go 实践示例

```go
// ❌ 特殊情况处理 - 坏品味
func removeNode(list *Node, target *Node) {
    if list == target {
        list = list.Next  // 特殊情况：删除头节点
        return
    }

    prev := list
    for prev.Next != nil {
        if prev.Next == target {
            prev.Next = prev.Next.Next
            return
        }
        prev = prev.Next
    }
}

// ✅ 消除特殊情况 - 好品味
func removeNode(indirect **Node, target *Node) {
    for *indirect != target {
        indirect = &(*indirect).Next
    }
    *indirect = target.Next  // 统一处理，无特殊情况
}
```

```go
// ❌ 深层嵌套 - 坏品味
func process(data *Data) error {
    if data != nil {
        if data.IsValid() {
            if data.HasPermission() {
                result := compute(data)
                if result != nil {
                    return save(result)
                }
            }
        }
    }
    return errors.New("failed")
}

// ✅ 早返回 - 好品味
func process(data *Data) error {
    if data == nil {
        return errors.New("nil data")
    }
    if !data.IsValid() {
        return errors.New("invalid data")
    }
    if !data.HasPermission() {
        return errors.New("no permission")
    }

    result := compute(data)
    if result == nil {
        return errors.New("compute failed")
    }

    return save(result)
}
```

---

## John Ousterhout: 复杂性管理 (Complexity Management)

> "Complexity is anything related to the structure of a software system that makes it hard to understand and modify."

### 15 条设计原则

| #   | 原则              | 说明                                              |
| --- | --------------- | ----------------------------------------------- |
| 1   | 复杂性是逐步增加的       | 必须处理小事情，小问题会累积成大问题                              |
| 2   | 能跑的代码是不够的       | Working code isn't enough                       |
| 3   | 持续小额投资改善设计      | Make continual small investments                |
| 4   | **模块应该深**       | 简单接口 + 强大功能                                     |
| 5   | 接口设计应简化常见用法     | 最常见的用法应该最简单                                     |
| 6   | **简单接口比简单实现重要** | 宁可复杂实现，不要复杂接口                                   |
| 7   | 通用模块更深          | General-purpose modules are deeper              |
| 8   | 通用和专用代码分开       | Separate general-purpose and special-purpose    |
| 9   | 不同层应有不同抽象       | Different layers, different abstractions        |
| 10  | **复杂性下沉**       | Pull complexity downward                        |
| 11  | **通过定义消除错误**    | Define errors out of existence                  |
| 12  | 设计两次            | 重要设计至少考虑两个方案再选择                                 |
| 13  | 注释描述代码中不明显的     | Comments for non-obvious things                 |
| 14  | 为阅读而设计          | Design for reading, not writing                 |
| 15  | 增量是抽象而非功能       | Increments should be abstractions, not features |

### 复杂性三大症状

| 症状 | 描述 |
|------|------|
| **变更放大** | 简单变更需要多处修改 |
| **认知负荷** | 开发者需要了解太多才能完成任务 |
| **未知的未知** | 不清楚哪些代码需要修改 |

### 模块深度

```
浅模块 (避免)              深模块 (追求)
┌─────────────────────┐    ┌─────┐
│  复杂接口           │    │简单 │ ← 接口
├─────────────────────┤    │接口 │
│  有限功能           │    ├─────┤
└─────────────────────┘    │     │
                           │强大 │ ← 实现
                           │功能 │
                           │     │
                           └─────┘
```

### Go 实践示例

```go
// ❌ 浅模块 - 接口复杂，功能有限
type FileReader interface {
    Open(path string, mode int, perm os.FileMode) error
    SetBufferSize(size int) error
    SetEncoding(enc string) error
    Read(buf []byte, offset int64, whence int) (int, error)
    Close() error
}

// ✅ 深模块 - 接口简单，功能强大
type FileReader interface {
    ReadFile(path string) ([]byte, error)  // 内部处理所有复杂性
}

// 实现中隐藏复杂性
func ReadFile(path string) ([]byte, error) {
    // 内部处理：打开、缓冲、编码、读取、关闭
    // 调用者无需知道这些细节
}
```

```go
// ❌ 通过异常处理错误
func GetUser(id string) (*User, error) {
    user, err := db.Find(id)
    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }
    return user, err
}

// ✅ 通过定义消除错误 (Define errors out of existence)
func GetOrCreateUser(id string, defaults User) *User {
    user, err := db.Find(id)
    if err == sql.ErrNoRows {
        return db.Create(defaults)  // 不返回错误，而是处理它
    }
    return user
}
```

---

## Google Code Review: 代码健康 (Code Health)

> "A CL that improves the overall code health of the system should not be delayed for perfection."

### 审查顺序

1. **设计** - 整体架构是否合理
2. **功能** - 是否正确实现需求
3. **复杂性** - 是否过度复杂
4. **测试** - 测试是否充分
5. **命名** - 命名是否清晰
6. **注释** - 注释是否有价值
7. **风格** - 是否符合规范
8. **文档** - 文档是否更新

### Small CLs 原则

| 规则 | 说明 |
|------|------|
| **100 行** | 合理的 CL 大小 |
| **1000 行** | 通常太大 |
| **One thing** | 一个 CL 应该是 one self-contained change |

---

## 危险信号清单 (Red Flags)

### Ousterhout 14 条危险信号

| # | 危险信号 | 描述 | 严重性 |
|---|----------|------|--------|
| 1 | **浅模块** | 接口复杂性 ≈ 实现复杂性 | 🔴 CRITICAL |
| 2 | **信息泄露** | 设计决策暴露在多个模块中 | 🔴 CRITICAL |
| 3 | 时间分解 | 代码结构基于操作顺序而非信息隐藏 | 🟡 WARNING |
| 4 | 过度暴露 | 常用功能需要了解罕用细节 | 🟡 WARNING |
| 5 | Pass-Through 方法 | 几乎只转发参数到另一个方法 | 🟡 WARNING |
| 6 | **代码重复** | 非平凡代码被反复复制 | 🔴 CRITICAL |
| 7 | 特殊/通用混合 | 专用代码和通用代码未分离 | 🟡 WARNING |
| 8 | 联合方法 | 两个方法强耦合，无法独立理解 | 🟡 WARNING |
| 9 | 注释重复代码 | 注释只是复述代码 | 🟡 WARNING |
| 10 | 实现污染接口 | 接口注释描述了不需要的实现细节 | 🟡 WARNING |
| 11 | 含糊的名称 | 名称不够精确，无法传达有用信息 | 🟡 WARNING |
| 12 | 难以命名 | 很难想出精确直观的名称 | 🟡 WARNING |
| 13 | **难以描述** | 完整文档需要很长 | 🔴 CRITICAL |
| 14 | **非显而易见的代码** | 行为或含义不容易理解 | 🔴 CRITICAL |

### 代码结构危险信号

| 信号 | 阈值 | 严重性 |
|------|------|--------|
| 嵌套层次 | > 3 层 | 🔴 CRITICAL |
| 函数长度 | > 100 行 | 🔴 CRITICAL |
| 局部变量 | > 10 个 | 🟡 WARNING |
| 无集中清理 | 多个退出点各自清理 | 🟡 WARNING |

### 异常处理危险信号

| 信号 | 描述 | 严重性 |
|------|------|--------|
| 防御性默认值 | `?? 0` 或 `\|\| ""` 掩盖问题 | 🟡 WARNING |
| 过多异常 | try-catch 比业务逻辑还多 | 🔴 CRITICAL |
| Fallback 代码 | 掩盖上游问题 | 🟡 WARNING |

---

## 经典语录 (Quotes)

### Linus Torvalds

| 场景 | 语录 |
|------|------|
| 防御性代码 | "Bad programmers worry about the code. Good programmers worry about data structures." |
| 深层嵌套 | "If you need more than 3 levels of indentation, you're screwed anyway." |
| 过度设计 | "Theory and practice sometimes clash. Theory loses. Every single time." |
| 特殊情况 | "Sometimes you can see a problem in a different way and rewrite it so that the special case goes away." |

### John Ousterhout

| 场景 | 语录 |
|------|------|
| 浅模块 | "Shallow modules don't help much in the battle against complexity." |
| 过多异常 | "The best way to eliminate exception handling complexity is to define your APIs so that there are no exceptions to handle." |
| Classitis | "Classes are good, so more classes are better - this is a mistake." |
| 设计 | "Design it twice. You'll end up with a much better result." |

### Google Code Review

| 场景 | 语录 |
|------|------|
| 评估变更 | "A CL that improves the overall code health of the system should not be delayed for perfection." |
| 过度工程 | "Encourage developers to solve the problem they know needs to be solved now, not the problem they speculate might need to be solved in the future." |
| 大 CL | "100 lines is usually a reasonable size for a CL, and 1000 lines is usually too large." |
