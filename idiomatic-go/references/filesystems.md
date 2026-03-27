# File Systems, Packaging & Deployment

## Table of Contents

- [io/fs Interface](#iofs-interface)
- [embed.FS](#embedfs)
- [Dev/Prod File Switching](#devprod-file-switching)
- [Config Loading Patterns](#config-loading-patterns)
- [Build & Deployment](#build--deployment)

## io/fs Interface

### Core Interfaces

```go
// fs.FS - Read-only file system
type FS interface {
    Open(name string) (File, error)
}

// fs.ReadFileFS - Direct file reading
type ReadFileFS interface {
    FS
    ReadFile(name string) ([]byte, error)
}

// fs.ReadDirFS - Directory listing
type ReadDirFS interface {
    FS
    ReadDir(name string) ([]DirEntry, error)
}
```

### Why io/fs?

| Benefit | Description |
|---------|-------------|
| Testability | Swap real FS with in-memory FS |
| Portability | Same code works with embed, OS, zip, etc. |
| Security | Read-only by design |
| Abstraction | No direct syscalls in business logic |

### Basic Usage

```go
import (
    "io/fs"
    "os"
)

// Accept fs.FS interface
func LoadConfig(fsys fs.FS, path string) (*Config, error) {
    data, err := fs.ReadFile(fsys, path)
    if err != nil {
        return nil, fmt.Errorf("read config: %w", err)
    }

    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }

    return &config, nil
}

// Use with real file system
func main() {
    config, err := LoadConfig(os.DirFS("."), "config.json")
    // ...
}
```

## embed.FS

### Basic Embedding

```go
import "embed"

// Embed single file
//go:embed config.json
var configData []byte

// Embed single file as string
//go:embed version.txt
var version string

// Embed directory
//go:embed templates/*
var templateFS embed.FS

// Embed multiple patterns
//go:embed static/* templates/*.html
var assetsFS embed.FS
```

### Reading Embedded Files

```go
//go:embed migrations/*.sql
var migrationsFS embed.FS

func runMigrations(db *sql.DB) error {
    entries, err := migrationsFS.ReadDir("migrations")
    if err != nil {
        return err
    }

    // Sort by name (001_init.sql, 002_users.sql, ...)
    sort.Slice(entries, func(i, j int) bool {
        return entries[i].Name() < entries[j].Name()
    })

    for _, entry := range entries {
        if entry.IsDir() {
            continue
        }

        data, err := migrationsFS.ReadFile("migrations/" + entry.Name())
        if err != nil {
            return fmt.Errorf("read %s: %w", entry.Name(), err)
        }

        if _, err := db.Exec(string(data)); err != nil {
            return fmt.Errorf("run %s: %w", entry.Name(), err)
        }
    }

    return nil
}
```

### Template with embed.FS

```go
import (
    "embed"
    "html/template"
)

//go:embed templates/*.html
var templateFS embed.FS

var templates = template.Must(
    template.ParseFS(templateFS, "templates/*.html"),
)

func renderPage(w http.ResponseWriter, name string, data any) error {
    return templates.ExecuteTemplate(w, name, data)
}
```

## Dev/Prod File Switching

### Build Tags Approach

```go
// assets_dev.go
//go:build dev

package assets

import "os"

// In dev, read from disk for hot reload
var FS = os.DirFS("./static")
```

```go
// assets_prod.go
//go:build !dev

package assets

import "embed"

//go:embed static/*
var FS embed.FS
```

```go
// Usage
package main

import "myapp/assets"

func main() {
    data, _ := fs.ReadFile(assets.FS, "static/index.html")
    // ...
}
```

```bash
# Development
go run -tags dev .

# Production
go build .  # No tag, uses embed
```

### Runtime Switch Approach

```go
//go:embed static/*
var embeddedFS embed.FS

type Config struct {
    DevMode bool
}

func GetFS(cfg Config) fs.FS {
    if cfg.DevMode {
        return os.DirFS("./static")
    }
    return embeddedFS
}
```

### Sub-directory fs.Sub

```go
//go:embed web/dist/*
var webFS embed.FS

func main() {
    // Get subdirectory as root
    distFS, err := fs.Sub(webFS, "web/dist")
    if err != nil {
        log.Fatal(err)
    }

    // Now "index.html" instead of "web/dist/index.html"
    http.Handle("/", http.FileServer(http.FS(distFS)))
}
```

## Config Loading Patterns

### FS-Agnostic Config Loader

```go
type ConfigLoader struct {
    fsys fs.FS
}

func NewConfigLoader(fsys fs.FS) *ConfigLoader {
    return &ConfigLoader{fsys: fsys}
}

func (l *ConfigLoader) Load(path string, v any) error {
    data, err := fs.ReadFile(l.fsys, path)
    if err != nil {
        return fmt.Errorf("read %s: %w", path, err)
    }

    ext := filepath.Ext(path)
    switch ext {
    case ".json":
        return json.Unmarshal(data, v)
    case ".yaml", ".yml":
        return yaml.Unmarshal(data, v)
    case ".toml":
        return toml.Unmarshal(data, v)
    default:
        return fmt.Errorf("unsupported format: %s", ext)
    }
}

// Usage
loader := NewConfigLoader(os.DirFS("/etc/myapp"))
var config AppConfig
if err := loader.Load("config.yaml", &config); err != nil {
    log.Fatal(err)
}
```

### Testing with fstest

```go
import "testing/fstest"

func TestConfigLoader(t *testing.T) {
    // Create in-memory filesystem
    mockFS := fstest.MapFS{
        "config.json": &fstest.MapFile{
            Data: []byte(`{"host": "localhost", "port": 8080}`),
        },
        "other.yaml": &fstest.MapFile{
            Data: []byte("host: localhost\nport: 8080"),
        },
    }

    loader := NewConfigLoader(mockFS)

    var config struct {
        Host string `json:"host" yaml:"host"`
        Port int    `json:"port" yaml:"port"`
    }

    if err := loader.Load("config.json", &config); err != nil {
        t.Fatal(err)
    }

    if config.Host != "localhost" {
        t.Errorf("got host %s, want localhost", config.Host)
    }
}
```

## Build & Deployment

### Build Flags for Version Info

```go
// main.go
package main

var (
    version   = "dev"
    commit    = "unknown"
    buildTime = "unknown"
)

func main() {
    fmt.Printf("Version: %s\nCommit: %s\nBuilt: %s\n",
        version, commit, buildTime)
}
```

```bash
# Build with version info
go build -ldflags "-X main.version=1.0.0 \
                   -X main.commit=$(git rev-parse --short HEAD) \
                   -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### Minimal Docker Build

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server

# Runtime stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Static Binary

```bash
# Fully static binary (no libc)
CGO_ENABLED=0 go build -ldflags="-s -w" -o app

# With version stripping (smaller)
go build -ldflags="-s -w" -o app

# UPX compression (optional, slower startup)
upx --best app
```

### Cross-Compilation

```bash
# Linux
GOOS=linux GOARCH=amd64 go build -o app-linux

# macOS ARM
GOOS=darwin GOARCH=arm64 go build -o app-mac-arm

# Windows
GOOS=windows GOARCH=amd64 go build -o app.exe
```

## Best Practices

| Practice | Reason |
|----------|--------|
| Accept `fs.FS` in functions | Enables testing and flexibility |
| Use `embed` for static assets | Single binary deployment |
| Use build tags for dev/prod | Hot reload in dev, embedded in prod |
| Use `fs.Sub` for subdirs | Cleaner paths |
| Test with `fstest.MapFS` | No disk I/O in tests |
