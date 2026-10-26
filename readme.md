## arkkvm-mcp install 

 npx --yes --package arkkvm-mcp@latest arkkvm_mcp

# arkkvm-mcp

Go implementation of the ArkKVM local MCP server — a full-parity rewrite of
[`arkkvm-mcp`](../arkkvm-mcp) (TypeScript). Same stdio JSON-RPC transport,
same tools, same behavior, error codes, and response shapes; a single
static ~13 MB binary instead of a ~90 MB Node SEA bundle.

## Build & install

Requires Go ≥ 1.25.

```bash
./scripts/build.sh                     # all targets: darwin/linux/windows, arm64+amd64
./scripts/build.sh darwin-arm64        # one target
go build -o bin/arkkvm-mcp ./cmd/arkkvm-mcp   # quick local build
```

Binaries land in `bin/arkkvm-mcp-<os>-<arch>[.exe]`; the host platform also
gets an unversioned `bin/arkkvm-mcp` alias (what Cursor-style `mcp.json`
files usually point at).

Runtime dependencies on the controller machine: `ffmpeg` (HDMI decode) and
`tesseract` (local OCR, configurable via `ARKKVM_OCR_COMMAND`).

## Client configuration

```json
{
  "mcpServers": {
    "arkkvm": {
      "command": "/path/to/arkkvm-mcp/bin/arkkvm-mcp"
    }
  }
}
```

## Environment variables

| Variable | Purpose |
|---|---|
| `ARKKVM_URL` | Device base URL (device LAN mode) |
| `ARKKVM_AGENT_COOKIE` / `ARKKVM_COOKIE` | Agent-role session cookie |
| `ARKKVM_AGENT_CSRF_TOKEN` / `ARKKVM_CSRF_TOKEN` | Agent CSRF token |
| `ARKKVM_AGENT_PASSWORD` / `ARKKVM_PASSWORD` | Device password (agent login) |
| `ARKKVM_ICE_SERVERS` | ICE servers as a JSON array |
| `ARKKVM_CLOUD_URL` / `ARKKVM_CLOUD_TOKEN` / `ARKKVM_CLOUD_DEVICE_ID` | Cloud relay mode (all three together) |
| `ARKKVM_CLOUD_APP_URL` | Login-page origin override |
| `ARKKVM_OCR_COMMAND` | OCR command (default `tesseract`) |
| `ARKKVM_OCR_LANG` | OCR language (default `eng+chi_sim`) |
| `ARKKVM_LOG_LEVEL` | `error`/`warn`/`info`/`debug` |
| `ARKKVM_LOG_FILE` | Optional append-only log file |

Everything can also be set at runtime through `kvm_configure` (in-memory
only). The only file the server writes is
`~/.config/arkkvm-mcp/cloud-token.json` (0600), holding the cloud token from
`kvm_cloud_login persist=true` plus the picked device target.

### Cloud regions

| Region | Cloud API (`cloudUrl`) | Login page (`appUrl`) |
|---|---|---|
| Global | `https://api.arkkvm.com` | `https://app.arkkvm.com` |
| China | `https://api.arkkvm.cn` | `https://app.arkkvm.cn` |

`api.<domain>` clouds derive `app.<domain>` automatically, so
`api.arkkvm.cn` opens `app.arkkvm.cn`. Pass `appUrl` (or set
`ARKKVM_CLOUD_APP_URL`) only when the login page lives elsewhere.

## Tools (30)

device_status · device_capabilities · kvm_configure · kvm_cloud_login ·
kvm_rpc (read-only methods including getControlState) · kvm_keyboard ·
kvm_mouse · kvm_wheel · kvm_get_control · kvm_request_control ·
kvm_deny_control · kvm_release_control ·
kvm_key_sequence · kvm_ip_forward_usb · kvm_ip_forward_host ·
hdmi_screenshot · hdmi_ocr · hdmi_ocr_stream · hdmi_frame_compare ·
hdmi_wait_for_change · hdmi_wait_for_stable · hdmi_video_monitor ·
hdmi_wait_for_text · kvm_health_snapshot · kvm_virtual_media ·
kvm_storage_upload · kvm_storage_delete · kvm_atx_power · kvm_virtual_usb ·
kvm_device_settings

Failed structured-tool calls return `structuredContent` with a stable error
code (`DEVICE_NOT_CONFIGURED`, `DEVICE_AUTH_REQUIRED`, `CONNECTION_*`,
`AGENT_SLOT_OCCUPIED`, `RPC_METHOD_NOT_EXPOSED`, `RPC_DEVICE_ERROR`,
`FT_VOLUME_NOT_MOUNTED`, `FT_FILE_EXISTS`, `UPLOAD_INCOMPLETE`,
`SOURCE_FILE_INVALID`, `INVALID_ARGUMENT`, `INTERNAL`) plus an actionable
hint. The MCP never auto-retries: the Agent decides whether to call again.

## Architecture

```
cmd/arkkvm-mcp        entry point (signals, stdio server)
internal/
  config/             in-memory settings store, env seeding, token file
  protocol/           device RPC surface, filename/path sanitizers
  toolerr/            result envelope, error classification, connection errors
  hid/                USB HID usage table, key-sequence planning
  textmatch/          OCR literal text matching (no regex, no ReDoS)
  signaling/          signaling frame shapes (device + cloud relay)
  webrtcsession/      pion/webrtc state machine: signaling WS, rpc/input
                      DataChannels, generation guard, single teardown path
  device/             DeviceClient: HTTP status/login, uploads, HDMI capture
  media/              H264 RTP → ffmpeg → PNG/gray8 frames, pixel diffing
  ocr/                tesseract wrapper
  cloudlogin/         loopback OIDC login, grant-code exchange, cloud devices
  forwarding/         TCP/UDP ↔ ip_forward_<uuid> DataChannel bridges
  upload/             upload pump constants (16 KiB chunks, ack-stall policy)
  settings/           kvm_device_settings actions
  server/             MCP server: 31 tool registrations
```

Key transport facts (shared with the TS version):

- WebRTC H264-only video negotiation; `rpc` and `input` ordered DataChannels;
  60 s signaling handshake, 25 ms channel-open polling, 2.5 s
  `disconnected` grace, teardown on RPC timeout.
- Cloud relay: signaling via `{cloudUrl}/webrtc/signaling/client?id=…&token=…`
  with `Authorization: Bearer`; TURN config from `POST /webrtc/ice_config`.
- Storage uploads: `startStorageFileUpload` / `startFtFsUpload` RPC, dedicated
  `upload_<uuid>` / `ft_upload_<uuid>` channels, 16 KiB chunks, resume from
  the acknowledged offset, ack-stall timeout 60 s + 1 s/MB capped at 10 min.
- ffmpeg decoder stays open across frames with `-probesize 32
  -analyzeduration 0 -fflags nobuffer` so the first frame does not wait for
  a 5 MB probe fill.

## Tests

```bash
go test ./...                                # unit tests (pure layers)
go build -o bin/arkkvm-mcp-darwin-arm64 ./cmd/arkkvm-mcp
python3 scripts/integration_test.py bin/arkkvm-mcp-darwin-arm64
```

The integration driver speaks stdio JSON-RPC to the built binary (per the
workspace MCP-testing rules): it runs with a scratch `HOME` (so the real
`~/.config/arkkvm-mcp/cloud-token.json` is never visible) and all `ARKKVM_*`
variables cleared, verifies `initialize` / `tools/list` (31 tools) /
`tools/call` behavior and exact error strings, then points `device_status`
at a mock device HTTP server. Real-device end-to-end (WebRTC media, HID,
cloud relay) needs live hardware.

## Not migrated from the TS version

- `pairing=true` on `kvm_cloud_login` (deprecated; the parameter is accepted
  for compatibility but only re-enables the legacy deviceId-in-URL flow).
- The SSH remote-terminal client (`remote_terminal_<uuid>`) — implemented in
  the TS DeviceClient but never registered as an MCP tool.
- Node SEA packaging (replaced by `GOOS/GOARCH` cross-compilation).
