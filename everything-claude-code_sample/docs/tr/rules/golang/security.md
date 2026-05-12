---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go Güvenlik

> Bu dosya [common/security.md](everything-claude-code_sample/docs/tr/rules/common/security.md) dosyasını Go'ya özgü içerikle genişletir.

## Secret Yönetimi

```go
apiKey := os.Getenv("OPENAI_API_KEY")
if apiKey == "" {
    log.Fatal("OPENAI_API_KEY not configured")
}
```

## Güvenlik Taraması

- Statik güvenlik analizi için **gosec** kullan:
  ```bash
  gosec ./...
  ```

## Context & Timeout'lar

Timeout kontrolü için daima `context.Context` kullan:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```
