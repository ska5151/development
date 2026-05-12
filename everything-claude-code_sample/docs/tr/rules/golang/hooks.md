---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go Hooks

> Bu dosya [common/hooks.md](everything-claude-code_sample/docs/tr/rules/common/hooks.md) dosyasını Go'ya özgü içerikle genişletir.

## PostToolUse Hooks

`~/.claude/settings.json` içinde yapılandır:

- **gofmt/goimports**: Edit'ten sonra `.go` dosyalarını otomatik formatla
- **go vet**: `.go` dosyalarını düzenledikten sonra statik analiz çalıştır
- **staticcheck**: Değiştirilen paketlerde genişletilmiş statik kontroller çalıştır
