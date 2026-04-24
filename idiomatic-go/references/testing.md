# Testing & Quality Gates

## Best Practices Summary

- Test behavior and contract, not implementation detail
- Use named subtests for scenario coverage
- Use `t.Parallel()` only when tests are truly isolated
- Run race detection for concurrent code
- Use `goleak` when goroutine lifecycle is part of correctness
- Keep benchmarks and examples executable and current

## Table of Contents

- [Test Fundamentals](#test-fundamentals)
- [Testify Essentials](#testify-essentials)
- [Table-Driven Tests](#table-driven-tests)
- [Parallel Testing](#parallel-testing)
- [Subtests](#subtests)
- [Fuzzing](#fuzzing)
- [Mocking & Interfaces](#mocking--interfaces)
- [Test Fixtures](#test-fixtures)
- [Coverage & Benchmarks](#coverage--benchmarks)
- [Test Suite Pattern (testify/suite)](#test-suite-pattern-testifysuite)

## Test Fundamentals

### Test File Structure

```go
// user_service_test.go
package user_test  // External test package (black-box)

import (
    "testing"

    "myapp/user"  // Package under test
)

func TestUserService_Create(t *testing.T) {
    // ...
}
```

### Naming Conventions

| Pattern | Example | Use For |
|---------|---------|---------|
| `TestXxx` | `TestUserCreate` | Unit tests |
| `TestType_Method` | `TestUserService_Create` | Method tests |
| `Test_helper` | `Test_validateEmail` | Internal helpers |
| `BenchmarkXxx` | `BenchmarkUserCreate` | Benchmarks |
| `FuzzXxx` | `FuzzParseJSON` | Fuzz tests |
| `ExampleXxx` | `ExampleUserCreate` | Documentation |

### Basic Test Structure

```go
func TestUserService_Create(t *testing.T) {
    // Arrange
    svc := NewUserService(mockRepo)
    input := CreateUserInput{Email: "test@example.com"}

    // Act
    user, err := svc.Create(context.Background(), input)

    // Assert
    require.NoError(t, err)
    require.NotNil(t, user)
    assert.Equal(t, input.Email, user.Email)
}
```

## Testify Essentials

### Add Dependency

```bash
go get github.com/stretchr/testify
go mod tidy
```

### assert vs require

- `assert.*`: 断言失败后继续执行，适合一次验证多个字段。
- `require.*`: 断言失败立即终止当前测试，适合前置条件（例如 `err == nil`、返回对象非空）。

```go
import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCreateUser(t *testing.T) {
    user, err := CreateUser("test@example.com")

    require.NoError(t, err)
    require.NotNil(t, user)

    assert.Equal(t, "test@example.com", user.Email)
    assert.NotZero(t, user.ID)
}
```

### High-Value Assertions

```go
func TestResponse(t *testing.T) {
    gotJSON := `{"age":30,"name":"john"}`
    wantJSON := `{"name":"john","age":30}`

    assert.JSONEq(t, wantJSON, gotJSON)
    assert.ElementsMatch(t, []int{1, 2, 2}, []int{2, 1, 2})
}

func TestErrors(t *testing.T) {
    err := fmt.Errorf("query failed: %w", ErrNotFound)

    assert.Error(t, err)
    assert.ErrorIs(t, err, ErrNotFound)
    assert.ErrorContains(t, err, "query failed")
}
```

### Eventually / Never (Async Assertions)

```go
func TestWorkerReady(t *testing.T) {
    var ready atomic.Bool

    go func() {
        time.Sleep(50 * time.Millisecond)
        ready.Store(true)
    }()

    assert.Eventually(t, ready.Load, time.Second, 10*time.Millisecond)
    assert.Never(t, func() bool { return false }, 100*time.Millisecond, 10*time.Millisecond)
}
```

> [!warning]
> `require.*` 必须在测试 goroutine 中调用（通常就是当前测试所在 goroutine）。

## Table-Driven Tests

### Basic Table Test

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid simple", "user@example.com", true},
        {"valid with subdomain", "user@sub.example.com", true},
        {"invalid no at", "userexample.com", false},
        {"invalid no domain", "user@", false},
        {"empty", "", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

### With Error Checking

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    *Config
        wantErr bool
    }{
        {
            name:  "valid json",
            input: `{"host": "localhost", "port": 8080}`,
            want:  &Config{Host: "localhost", Port: 8080},
        },
        {
            name:    "invalid json",
            input:   `{invalid}`,
            wantErr: true,
        },
        {
            name:    "empty",
            input:   "",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseConfig([]byte(tt.input))

            if tt.wantErr {
                require.Error(t, err)
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

## Parallel Testing

### Basic Parallel

```go
func TestSlowOperation(t *testing.T) {
    t.Parallel()  // Mark test as parallel

    // Test body...
}
```

### Parallel with Table Tests

```go
func TestFetch(t *testing.T) {
    tests := []struct {
        name string
        url  string
        want int
    }{
        {"google", "https://google.com", 200},
        {"github", "https://github.com", 200},
        {"invalid", "https://invalid.invalid", 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()  // Each subtest runs in parallel

            resp, err := http.Get(tt.url)
            if tt.want == 0 {
                if err == nil {
                    t.Error("expected error")
                }
                return
            }

            if resp.StatusCode != tt.want {
                t.Errorf("got status %d, want %d", resp.StatusCode, tt.want)
            }
        })
    }
}
```

### Parallel Safety

```go
func TestConcurrentMap(t *testing.T) {
    t.Parallel()

    m := NewConcurrentMap()

    // Use t.Cleanup for shared resources
    t.Cleanup(func() {
        m.Close()
    })

    t.Run("write", func(t *testing.T) {
        t.Parallel()
        m.Set("key", "value")
    })

    t.Run("read", func(t *testing.T) {
        t.Parallel()
        _ = m.Get("key")
    })
}
```

> [!tip]
> `suite.Suite` 适合做结构化测试组织，但默认不建议在 suite 的测试方法里直接并行化（容易引入共享状态问题）。并行场景优先使用普通 `TestXxx` + `t.Run` + `t.Parallel()`。

## Subtests

### Grouping Related Tests

```go
func TestUserService(t *testing.T) {
    svc := setupService(t)

    t.Run("Create", func(t *testing.T) {
        t.Run("success", func(t *testing.T) {
            // ...
        })
        t.Run("duplicate email", func(t *testing.T) {
            // ...
        })
    })

    t.Run("Update", func(t *testing.T) {
        t.Run("success", func(t *testing.T) {
            // ...
        })
        t.Run("not found", func(t *testing.T) {
            // ...
        })
    })
}
```

### Running Specific Subtests

```bash
# Run all tests in TestUserService
go test -run TestUserService

# Run only Create subtests
go test -run TestUserService/Create

# Run specific subtest
go test -run "TestUserService/Create/success"
```

## Fuzzing

### Basic Fuzz Test

```go
func FuzzParseJSON(f *testing.F) {
    // Add seed corpus
    f.Add([]byte(`{"name": "test"}`))
    f.Add([]byte(`{}`))
    f.Add([]byte(`{"count": 42}`))

    f.Fuzz(func(t *testing.T, data []byte) {
        var result map[string]any

        // Should never panic
        _ = json.Unmarshal(data, &result)

        // If parsing succeeds, re-encoding should work
        if err := json.Unmarshal(data, &result); err == nil {
            _, err := json.Marshal(result)
            if err != nil {
                t.Errorf("failed to re-marshal valid JSON: %v", err)
            }
        }
    })
}
```

### Fuzz with Multiple Args

```go
func FuzzConcat(f *testing.F) {
    f.Add("hello", "world")
    f.Add("", "test")
    f.Add("test", "")

    f.Fuzz(func(t *testing.T, a, b string) {
        result := Concat(a, b)

        if len(result) != len(a)+len(b) {
            t.Errorf("length mismatch: got %d, want %d",
                len(result), len(a)+len(b))
        }

        if !strings.HasPrefix(result, a) {
            t.Error("result should start with first arg")
        }
    })
}
```

### Running Fuzz Tests

```bash
# Run fuzz test for 30 seconds
go test -fuzz FuzzParseJSON -fuzztime 30s

# Run with specific corpus
go test -fuzz FuzzParseJSON -fuzztime 1m

# View failing inputs
# Saved in testdata/fuzz/FuzzParseJSON/
```

## Mocking & Interfaces

### Interface-Based Mocking

```go
// Define interface for dependencies
type UserRepository interface {
    Find(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Mock implementation
type mockUserRepo struct {
    users map[string]*User
    err   error
}

func (m *mockUserRepo) Find(ctx context.Context, id string) (*User, error) {
    if m.err != nil {
        return nil, m.err
    }
    user, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

func (m *mockUserRepo) Save(ctx context.Context, user *User) error {
    if m.err != nil {
        return m.err
    }
    m.users[user.ID] = user
    return nil
}

// Test using mock
func TestUserService_GetUser(t *testing.T) {
    repo := &mockUserRepo{
        users: map[string]*User{
            "1": {ID: "1", Email: "test@example.com"},
        },
    }

    svc := NewUserService(repo)
    user, err := svc.GetUser(context.Background(), "1")

    require.NoError(t, err)
    require.NotNil(t, user)
    assert.Equal(t, "test@example.com", user.Email)
}
```

### Using testify/mock (Recommended Patterns)

```go
import "github.com/stretchr/testify/mock"

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) Find(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepo) Save(ctx context.Context, user *User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

func TestWithMock(t *testing.T) {
    repo := new(MockUserRepo)

    repo.On("Find", mock.Anything, "1").
        Return(&User{ID: "1", Email: "test@example.com"}, nil).
        Once()

    svc := NewUserService(repo)
    user, err := svc.GetUser(context.Background(), "1")

    require.NoError(t, err)
    require.NotNil(t, user)
    assert.Equal(t, "1", user.ID)
    assert.Equal(t, "test@example.com", user.Email)

    repo.AssertExpectations(t)
}
```

### Advanced Mock Controls

```go
func TestProcessor_OrderAndOptionalCalls(t *testing.T) {
    m := new(MockService)

    initCall := m.On("Init").Return(nil).Once()
    m.On("Connect").Return(nil).NotBefore(initCall).Once()
    m.On("Warmup", mock.Anything).Return(nil).Maybe() // optional call

    err := RunProcess(m)

    require.NoError(t, err)
    m.AssertExpectations(t)
}

func TestProcessor_InOrder(t *testing.T) {
    m := new(MockService)

    mock.InOrder(
        m.On("Init").Return(nil).Once(),
        m.On("Connect").Return(nil).Once(),
        m.On("Process", mock.Anything).Return(nil).Once(),
    )

    err := RunProcess(m)

    require.NoError(t, err)
    m.AssertExpectations(t)
}
```

> [!note]
> 常用匹配器：`mock.Anything`、`mock.AnythingOfType("string")`、`mock.MatchedBy(func(T) bool { ... })`。

## Test Fixtures

### Setup and Teardown

```go
func TestMain(m *testing.M) {
    // Global setup
    setup()

    code := m.Run()

    // Global teardown
    teardown()

    os.Exit(code)
}

// Per-test setup with t.Cleanup
func TestWithDB(t *testing.T) {
    db := setupTestDB(t)
    t.Cleanup(func() {
        db.Close()
    })

    // Test using db...
}
```

### Test Helpers

```go
func setupService(t *testing.T) *UserService {
    t.Helper() // Mark as helper for better error locations

    repo := &mockUserRepo{users: make(map[string]*User)}
    return NewUserService(repo)
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    require.NoError(t, err)
}

func assertEqual[T comparable](t *testing.T, got, want T) {
    t.Helper()
    assert.Equal(t, want, got)
}
```

### Golden Files

```go
func TestRender(t *testing.T) {
    got := Render(testInput)

    golden := filepath.Join("testdata", t.Name()+".golden")

    if *update {  // -update flag
        os.WriteFile(golden, []byte(got), 0644)
    }

    want, err := os.ReadFile(golden)
    if err != nil {
        t.Fatal(err)
    }

    if got != string(want) {
        t.Errorf("output mismatch.\ngot:\n%s\nwant:\n%s", got, want)
    }
}

var update = flag.Bool("update", false, "update golden files")
```

## Coverage & Benchmarks

### Running with Coverage

```bash
# Basic coverage
go test -cover ./...

# Atomic mode for concurrency-safe coverage in parallel tests
go test -covermode=atomic -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
go tool cover -html=coverage.out
```

### Coverage Thresholds

```bash
# Fail if coverage < 80%
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | grep total | awk '{print $3}' | \
    awk -F'%' '{if ($1 < 80) exit 1}'
```

### Benchmarks

```go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData()

    b.ResetTimer()
    for b.Loop() {
        Process(data)
    }
}

func BenchmarkProcess_Parallel(b *testing.B) {
    data := generateTestData()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Process(data)
        }
    })
}

func BenchmarkProcess_Sizes(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := generateTestDataOfSize(size)
            b.ResetTimer()

            for b.Loop() {
                Process(data)
            }
        })
    }
}
```

### Running Benchmarks

```bash
# Run all benchmarks
go test -bench=. ./...

# With memory allocation stats
go test -bench=. -benchmem ./...

# Specific benchmark
go test -bench=BenchmarkProcess -benchmem

# Stabilize benchmark runs (no test execution, fixed time)
go test -run=^$ -bench=. -benchmem -benchtime=1s -count=5 ./...

# Compare benchmarks
go test -bench=. -count=10 > old.txt
# make changes
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt
```

## Test Suite Pattern (testify/suite)

```go
import (
    "testing"

    "github.com/stretchr/testify/suite"
)

type UserServiceSuite struct {
    suite.Suite
    svc  *UserService
    repo *MockUserRepo
}

func (s *UserServiceSuite) SetupTest() {
    s.repo = new(MockUserRepo)
    s.svc = NewUserService(s.repo)
}

func (s *UserServiceSuite) TearDownTest() {
    s.repo.AssertExpectations(s.T())
}

func (s *UserServiceSuite) TestGetUser() {
    s.repo.On("Find", mock.Anything, "1").
        Return(&User{ID: "1", Email: "test@example.com"}, nil).
        Once()

    user, err := s.svc.GetUser(context.Background(), "1")

    s.Require().NoError(err)
    s.Require().NotNil(user)
    s.Equal("test@example.com", user.Email)
}

func TestUserServiceSuite(t *testing.T) {
    suite.Run(t, new(UserServiceSuite))
}
```

## Goroutine Leak Detection

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Or per-test
func TestNoLeak(t *testing.T) {
    defer goleak.VerifyNone(t)

    // Test code...
}
```
