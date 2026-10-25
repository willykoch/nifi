# File Gateway Upload

Uploads PDFs from a mounted inbound folder to the file gateway and logs each result to a CSV file.

## Flow

```text
GetFile → CryptographicHashContent → UpdateAttribute → InvokeHTTP (PUT)
                                                       └─ Response → EvaluateJsonPath → AttributesToCSV → ExecuteGroovyScript (append CSV)
```

1. `GetFile` picks up PDFs from `/data/inbound` every 10 seconds and keeps the source files.
2. `CryptographicHashContent` computes the SHA-256 of the PDF bytes.
3. `InvokeHTTP` PUTs the unchanged PDF to `#{gateway.put.url}<filename>`, with the filename percent-encoded.
4. The JSON response is parsed for `URI`, and one CSV row is appended to the log.

## Parameters

Set these in the "File Gateway Upload Parameters" parameter context:

| Parameter | Value | Sensitive |
|---|---|---|
| `gateway.username` | `0101` | no |
| `gateway.password` | gateway password, entered in NiFi | yes |
| `gateway.put.url` | `https://void.0101.no/fileapi/` (keep the trailing slash) | no |

The CSV path is hard-coded in the `Append CSV Log` script as `/data/inbound/log/file-gateway-uploads.csv`.

## Container setup

The NiFi container needs these mounts:

```yaml
- /opt/nifi-stack/data/inbound:/data/inbound
```

The folder must be owned by UID 1000 (the `nifi` user in the container). The CSV log is written to `/data/inbound/log/`. Create that folder and add the header row once before the first run:

```
filename,sha256,http_status,http_message,uri
```

## Limitations

- The gateway API rejects non-ASCII filenames (for example `ø`, `å`, `æ`). Rename those files before they are uploaded.
- A PUT to an existing name overwrites the file on the gateway.
- Archiving or moving source files after upload is not implemented; `GetFile` keeps them.
