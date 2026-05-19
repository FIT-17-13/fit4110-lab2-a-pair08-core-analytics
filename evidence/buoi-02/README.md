# Evidence Buổi 02

Thư mục này lưu bằng chứng Lab 02.

Cần có:

```text
evidence/buoi-02/
  README.md
  checklist.md
  known-issues.md
  spectral-report.txt
  tool-versions.txt
  git-log.txt
  mock-screenshots/
    req-01-*.png
    req-02-*.png
    req-03-*.png
    req-04-*.png
    req-05-*.png
```

## Cách sinh report Spectral

```bash
./scripts/collect_session02_evidence.sh
```

Windows:

```powershell
.\scripts\collect_session02_evidence.ps1
```

## Ảnh mock server

Lab 02 chưa yêu cầu Postman. Minh chứng nên là ảnh chụp Terminal/PowerShell khi chạy `curl` tới Prism mock server.

Mỗi ảnh cần thể hiện:

- lệnh `curl` trong Terminal/PowerShell;
- status code;
- response body;
- URL `http://localhost:4010/...`.


## Chạy mock Lab 02

Cài công cụ nếu máy chưa có:

```powershell
npm install -g @stoplight/spectral-cli @stoplight/prism-cli

Chạy Spectral lint và lưu report:
npx spectral lint openapi.yaml --ruleset .\campus-spectral.yaml --format text > evidence/buoi-02/spectral-report.txt

Chạy Prism mock server ở port 4010:
npx prism mock openapi.yaml --port 4010

Done. Prism is listening on http://127.0.0.1:4010

Mở terminal PowerShell mới và test các request mẫu:
  1.
Invoke-RestMethod -Uri "http://127.0.0.1:4010/health" -Method GET
  2.
curl.exe -X POST "http://127.0.0.1:4010/events/core" `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer demo-token" `
  --data-binary "@body-policy.json"
  3.
curl.exe -X POST "http://127.0.0.1:4010/events/core" `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer demo-token" `
  --data-binary "@body-alert-created.json"
  4.
curl.exe -X POST "http://127.0.0.1:4010/events/core" `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer demo-token" `
  --data-binary "@body-alert-resolved.json"
  5.
  Invoke-RestMethod `
  -Uri "http://127.0.0.1:4010/analytics/core/summary?from=2026-05-19T00:00:00Z&to=2026-05-19T23:59:59Z" `
  -Method GET `
  -Headers @{ Authorization = "Bearer demo-token" }