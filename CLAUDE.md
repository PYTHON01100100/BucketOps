# CLAUDE.md

This file guides Claude Code when working in this repository.

## Project

**BucketOps** is a user-friendly tool, written in Go, for managing Alibaba Cloud OSS (Object Storage Service) buckets. It is a single static binary with **two interfaces** built on one shared core:

| Interface | Command | For |
|---|---|---|
| **TUI** (interactive) | `bucketops` (no arguments) or `bucketops ui` | Browsing buckets, reading files, picking folders to upload, switching profiles, all with the keyboard |
| **CLI** (scriptable) | `bucketops <command> ...` | Automation, scripts, CI, one-off commands |

Both interfaces call the same packages in `internal/core`. **Never put OSS logic in the UI layers.** A feature added to the core should be usable from both.

Main features:

1. **Profiles:** switch in one step between accounts and regions, each with its own access key, secret and region. Profiles are shared with `ossutil`.
2. **Browse:** move through buckets and folders like a file manager.
3. **Cat:** open any object in a preview pane, or stream it to stdout.
4. **Upload:** pick specific local folders, or pick one directory and upload **all of its folders to the bucket root**.
5. **Download:** download a remote folder to a local directory.
6. **Bucket management:** list, create, inspect and delete buckets.

The repo is new. Only `LICENSE` (MIT) exists so far. Build to the spec below.

## Tech stack

- **Language:** Go 1.23+. The module path is `github.com/<owner>/bucketops`; confirm the owner with the user before running `go mod init`.
- **OSS SDK:** `github.com/aliyun/alibabacloud-oss-go-sdk-v2`, with packages `oss` and `oss/credentials`. Do not use the legacy `aliyun-oss-go-sdk` v1.
- **TUI:** Bubble Tea (`github.com/charmbracelet/bubbletea`), Bubbles (list, table, viewport, textinput, progress, spinner, help, key) and Lip Gloss for styling
- **CLI:** `github.com/spf13/cobra`
- **Config:** `gopkg.in/ini.v1`, which reads the INI format that ossutil uses. App state is stored as TOML with `github.com/BurntSushi/toml`.
- **Syntax highlighting:** `github.com/alecthomas/chroma/v2`
- **Clipboard:** `github.com/atotto/clipboard`
- **Tests:** the standard `testing` package with table-driven tests, plus `github.com/charmbracelet/x/exp/teatest` for the TUI. The OSS client sits behind an interface and is faked in tests; unit tests make no network calls.
- **Lint/format:** `gofmt`/`goimports` and `golangci-lint`
- **Release:** `goreleaser` builds for linux, darwin and windows on amd64 and arm64 with `CGO_ENABLED=0`

## Layout

```
cmd/bucketops/main.go        # wires cobra root; no args → TUI
internal/
  core/                      # UI-agnostic; must not import cobra, bubbletea or lipgloss; no fmt.Print
    config/                  # profile load/save/switch (reads and writes ~/.ossutilconfig), state.toml
    ossclient/               # builds *oss.Client from a profile; defines the OSSAPI interface used by everything
    buckets/                 # bucket CRUD
    browse/                  # list folders and objects under a prefix (paginated, delimiter "/")
    transfer/                # UploadFolders / UploadDirToRoot / DownloadFolder, worker pool, sync logic
    cat/                     # streaming and range reads (io.Reader based)
    keys/                    # pure helpers: local path <-> object key mapping
    events/                  # Progress/Result event types sent on channels or callbacks
    errs/                    # domain errors: ErrBucketNotFound, ErrAccessDenied, ErrWrongRegion, ErrInvalidCredentials
  cli/                       # cobra commands: parsing and rendering only
  tui/
    app.go                   # root model; routes messages to the active screen
    screens/                 # profiles, buckets, browser, preview, upload, transfers, help
    components/              # dirtree (multi-select), breadcrumb, statusbar, confirm dialog, toast
    styles/                  # lipgloss theme (dark and light)
    keys.go                  # key.Binding definitions, shared by the help view
```

- The core accepts a `context.Context` everywhere so every operation can be cancelled, for example with `x` in the TUI or Ctrl-C in the CLI.
- The core reports progress through `events.Progress` values. The CLI renders them as a text progress bar, and the TUI turns them into `tea.Msg` values through a channel-reading `tea.Cmd`.

## Profiles (credentials and region)

### Storage: compatible with ossutil

Profiles live in **`~/.ossutilconfig`**, the same file that [ossutil 2.0](https://www.alibabacloud.com/help/en/oss/developer-reference/install-ossutil) uses. A profile you create in ossutil works in BucketOps, and the reverse is also true.

```ini
[default]
accessKeyID = LTAI...
accessKeySecret = ...
region = cn-hangzhou

[profile prod-sg]
accessKeyID = LTAI...
accessKeySecret = ...
region = ap-southeast-1
endpoint = https://oss-ap-southeast-1.aliyuncs.com   ; optional, derived from region if missing
```

This format is verified against the [ossutil 2.0 configuration docs](https://www.alibabacloud.com/help/en/oss/developer-reference/configure-ossutil2):

- `[default]` is shorthand for `[profile default]`. Named profiles use `[profile NAME]`. Treat the two forms of the default as the same profile.
- `[buckets NAME]` sections map buckets to endpoints. Keep them, and use their endpoint for the matching bucket.
- Key names are **case-insensitive**, and ossutil accepts several spellings: `accessKeyID`, `access-key-id` and `access_key_id` all mean the same key, as do `stsToken`, `sts-token` and `sts_token`. Normalize names when reading. When writing, use the camelCase form: `accessKeyID`, `accessKeySecret`, `stsToken`, `region`, `endpoint`.
- The `mode` key selects the credential type. Support `AK` (the default) and `StsToken`. For other modes such as `RamRoleArn` or `EcsRamRole`, show a clear "not supported yet" message instead of failing obscurely.
- ossutil 1.x wrote a different format, with a `[Credentials]` section and no region. If you find that format, explain that it is from ossutil 1.x and offer to migrate it to a 2.0-style profile, asking for the region, which 2.0 requires for V4 signing. Never rewrite the file silently.
- `region` is **required**, because ossutil 2.0 and SDK v2 use V4 signatures.
- Other keys that ossutil 2.0 reads, such as `loglevel`, `read-timeout`, `connect-timeout`, `retry-times`, `output-format`, `addressing-style` and `language`, must be kept when writing. Use them only where they apply.
- BucketOps keeps its own settings in `~/.bucketops/state.toml`: the last active profile, recent buckets and recent local directories. It never stores secrets there.
- When writing `~/.ossutilconfig`, keep any keys and sections BucketOps doesn't know about. Write to a temporary file in the same directory, then `os.Rename` it over the original, so a crash can't corrupt it. Set the file mode to `0600`.
- `stsToken` is supported for temporary credentials. Use `credentials.NewStaticCredentialsProvider(id, secret, token)`.
- Build the client with `oss.LoadDefaultConfig().WithCredentialsProvider(...).WithRegion(...)`. Set `WithEndpoint` only when the profile has an endpoint. Use `WithUseInternalEndpoint(true)` for `--internal`.

### Resolution order

This follows ossutil 2.0's order: command-line options, then environment variables, then the config file.

- **Config file path:** `--config-file`, then `OSSUTIL_CONFIG_FILE`, then `~/.ossutilconfig`
- **Profile:** `--profile NAME`, then `OSSUTIL_PROFILE`, then the last active profile from `state.toml`, then `default`
- **Individual values:** command-line flags (`--region`, `--endpoint`) override the environment variables `OSS_ACCESS_KEY_ID`, `OSS_ACCESS_KEY_SECRET`, `OSS_SESSION_TOKEN`, `OSS_REGION` and `OSS_ENDPOINT`. Those override the profile's values.

Use the same environment variable names as ossutil, so an environment set up for ossutil works for BucketOps too. Don't invent `BUCKETOPS_*` names for these settings.

### Profile UX

- **CLI:**
  - `bucketops profile ls`: shows each profile's name, region and masked key (`LTAI****abcd`) and marks the active one with `*`
  - `bucketops profile add [NAME]`: interactive prompts; the secret is read with `golang.org/x/term` `ReadPassword`, so the input is hidden
  - `bucketops profile use NAME`
  - `bucketops profile edit NAME`
  - `bucketops profile rm NAME`
  - `bucketops profile test [NAME]`: calls `ListBuckets` and reports whether it succeeded, how long it took and the reason if it failed
- **TUI:** press `p` anywhere to open the profile switcher, a filterable `list.Model`. `Enter` switches profiles and reloads the bucket list. `a` adds, `e` edits and `d` deletes (with confirmation). The form has a filterable region picker with all OSS regions and a masked secret field (`textinput.EchoPassword`). Saving runs the connection test automatically.
- The active profile and region are **always visible**: in the TUI header, and in the first line of CLI output for destructive commands.

### Security

- **Never** log, print or display a secret. The UI masks it, and when editing, the secret field is empty; leaving it empty keeps the current value.
- Access keys are masked everywhere except the edit form. Give the profile type a `String()` method that masks the key, so `%v` can't leak it.
- Never commit credential files. Add `.env`, `*.ossutilconfig`, `state.toml` and `dist/` to `.gitignore`.
- The docs should recommend RAM users or STS with least-privilege policies, not root account keys.

## TUI design

The layout is two panes, like a file manager:

```
┌ BucketOps ─ profile: prod-sg ─ region: ap-southeast-1 ───────────────────┐
│ oss://my-bucket/reports/2026/                          [/] filter        │
├───────────────────────────────┬──────────────────────────────────────────┤
│ ..                            │ q1.csv  · 2.4 MB · 2026-04-01 · Standard │
│ 📁 jan/                        │──────────────────────────────────────────│
│ 📁 feb/                        │ date,region,amount                       │
│ 📄 q1.csv          2.4 MB     │ 2026-01-01,sg,1200                       │
│ 📄 summary.json    3 KB       │ ...  (first 64 KB, syntax highlighted)   │
├───────────────────────────────┴──────────────────────────────────────────┤
│ ↑↓ move  ⏎ open  c cat  u upload  d download  p profile  ? help  q quit  │
└──────────────────────────────────────────────────────────────────────────┘
```

### Navigation

- The start screen lists buckets for the active profile, showing region, creation date and storage class. `Enter` opens a bucket.
- `Enter`, `→` or `l` opens a folder. `Backspace`, `←` or `h` goes up. Arrow keys and vim-style keys both work.
- `/` filters the current listing as you type. `g` jumps to a path by typing `oss://bucket/prefix`.
- Listings load lazily, one page at a time, with `oss.NewListObjectsV2Paginator` and `Delimiter: "/"`. The next page is fetched when the cursor nears the end. Large prefixes must never freeze the UI.
- `r` refreshes. `s` cycles the sort order: name, size, then modified time.

### Cat and preview

- Moving the cursor onto a file shows a **live preview** in the right pane. The preview uses a range GET for only the first 64 KB and waits 150 ms after the cursor stops before loading (use `tea.Tick` and ignore results that arrive after the cursor has moved on).
- `c` or `Enter` on a file opens a **full-screen viewer** (`viewport.Model`). It loads more content as you scroll, has `/` search, `w` to toggle line wrapping and `y` to copy the key to the clipboard.
- Rendering depends on the file type:
  - Text, logs, CSV and TSV: CSV shows as an aligned table.
  - JSON: pretty-printed with `json.Indent`.
  - Code: syntax highlighted with chroma.
  - Images: show metadata only.
  - Binary: show a hex dump of the first bytes with `encoding/hex` `Dump`.
- Detect the type from the content type, the file extension and `http.DetectContentType` on the first bytes.
- The preview always shows size, content type, last-modified time, ETag and storage class.
- Archive-class objects show a message that they must be restored first, with an `R` action to restore them. Never fail silently.

### Upload flow

Press `u` in any bucket or folder. The upload **destination** is the folder currently open in the browser; at the bucket root, files go to the root.

1. **Pick a source mode:**
   - **Choose folders:** a local directory tree (the custom `components/dirtree`) with checkboxes. Press `Space` to select one or more folders, `a` to select all and `n` to select none. Each selected folder is uploaded as a folder, keeping its name.
   - **Whole directory → root:** pick one local directory. **Every folder and file inside it** is uploaded to the destination, without the parent directory's name.
2. **Preview step (required):** show the resulting remote keys as a tree, with file counts, total size, and how many files will be skipped because they are unchanged. Show a warning if any existing objects would be overwritten.
3. **Confirm**, then the transfer runs on the Transfers screen (press `t` to open it). The screen shows per-file and overall progress bars (`progress.Model`), speed and time remaining, with `x` to cancel through the context. The UI stays usable while transfers run.
4. Recent local directories are remembered in `state.toml` for next time.

### Download flow

Press `d` on a folder or file. A local directory picker opens with the last-used directory selected. A preview step like the upload one follows, then the transfer runs on the Transfers screen.

### General TUI rules

- Every OSS call runs in a `tea.Cmd`, which Bubble Tea runs in a goroutine. **Never block in `Update`.**
- Tag results with a request ID and ignore stale ones, for example after the user has navigated away.
- Errors show as a toast notification with a clear, human-readable message, such as "Access denied: profile prod-sg cannot list my-bucket." Press `L` to see details. Never panic; recover at the top level and restore the terminal.
- Destructive actions need a confirmation dialog, and deleting a folder requires typing the folder name.
- `?` shows the full help view (`help.Model`, generated from `keys.go`). The bottom bar always shows the most important keys for the current context.
- Support dark and light themes with `lipgloss.AdaptiveColor`, and respect `NO_COLOR`.
- The layout adapts to `tea.WindowSizeMsg` and is usable in an 80×24 terminal. On narrow widths, hide the preview pane.
- Use `tea.WithAltScreen()`. Mouse support is optional, with `tea.WithMouseCellMotion()`.

## CLI commands

```
bucketops                                     # launches the TUI
bucketops ui [oss://bucket/prefix]            # launches the TUI at a location

bucketops profile ls|add|use|edit|rm|test

bucketops bucket ls
bucketops bucket create <bucket> [--region R] [--storage-class Standard|IA|Archive]
bucketops bucket info <bucket>
bucketops bucket rm <bucket> [--force]

bucketops ls <bucket>[/prefix] [--recursive] [--long]
bucketops cat <bucket>/<key> [--range 0-1023] [--head N]
bucketops upload <local_path>... <bucket>[/prefix] [--dry-run] [--delete] [--exclude GLOB] [--workers N]
bucketops download <bucket>/<prefix> <local_dir> [--dry-run] [--workers N]
bucketops rm <bucket>/<key-or-prefix> [--recursive]

bucketops doctor                              # checks config, connectivity and clock skew; reports ossutil if installed
bucketops version
```

- `--config-file PATH`, `--profile NAME`, `--region R`, `--endpoint URL`, `--yes`, `--debug` and `--output table|json` are persistent flags on the root command.
- `oss://bucket/key` is accepted anywhere a `<bucket>/<key>` is expected.
- Shell completion comes from Cobra (`bucketops completion bash|zsh|fish|powershell`). Bucket and key names complete through `ValidArgsFunction`, which does a live lookup with a short timeout.
- If a required argument is missing and stdin is a terminal (`term.IsTerminal`), offer an interactive picker instead of failing. In scripts, fail with a clear usage error.
- `--output json` produces machine-readable output for `ls`, `bucket ls`, `profile ls` and transfer summaries.

## Folder conventions (important)

OSS has no real directories. A "folder" is a key prefix ending in `/`. `internal/core/keys` enforces these rules.

### Upload modes

| Mode | CLI | TUI | Local `./data/{a,b}/f.txt` → remote |
|---|---|---|---|
| **Folder** (keep name) | `upload ./data bkt` | "Choose folders" → `data` | `data/a/f.txt`, `data/b/f.txt` |
| **Several folders** | `upload ./data/a ./data/b bkt` | "Choose folders" → `a`, `b` | `a/f.txt`, `b/f.txt` |
| **Directory → root** (contents) | `upload ./data/ bkt` (trailing slash) | "Whole directory → root" | `a/f.txt`, `b/f.txt` |

- With a destination prefix such as `bkt/archive`, or when the browser is open at `archive/`, every mode puts the result under `archive/`.
- The key **never** includes the absolute or parent local path. For example, `home/emacs/...` is always wrong.

### Key rules

- Build keys with `filepath.Rel` followed by `filepath.ToSlash`, and join them with `path.Join`, not `filepath.Join`. Keys always use `/`, and Windows paths must also work.
- Never start a key with `/`. Collapse `//` and reject `..` segments.
- Keys are UTF-8. Keep the original filename case.
- Walk the local tree with `filepath.WalkDir`. Skip by default: `.git/`, `node_modules/`, `.DS_Store`, `Thumbs.db` and `*.tmp`. Extend the list with `--exclude` (glob) or in the TUI's upload options.
- Don't follow symlinks by default. Following them requires `--follow-symlinks`.
- Do not create zero-byte "directory marker" objects, except for empty local folders when `--keep-empty-dirs` is set.

### Download

- `download bkt/reports ./out` writes `./out/reports/...`, the mirror image of folder-mode upload.
- Before writing any file, check that the resolved local path stays inside the target directory: take `filepath.Rel` from the target and reject results that start with `..`. This blocks path traversal from malicious keys.
- Skip directory-marker keys that end in `/`. Create the local directories they imply.

## Transfer best practices

- **Large files:** use the SDK's `client.NewUploader` and `client.NewDownloader` managers, which handle multipart transfers. Use multipart for files at or above 100 MiB, with an 8 MiB default part size.
- **Resumable:** set `EnableCheckpoint: true` with `CheckpointDir: ~/.bucketops/checkpoints/` so interrupted transfers continue instead of restarting.
- **Concurrency:** use a worker pool across files (`errgroup` with `SetLimit`). The default is 8 workers. Don't let the errgroup cancel the batch on one file's error; collect per-file errors instead.
- **Skip unchanged files:** compare size, then CRC64 (OSS returns `x-oss-hash-crc64ecma`; compute the local value with `hash/crc64` and the ECMA table). Re-running an upload acts as a sync.
- **Integrity:** keep the SDK's CRC check turned on, which is the default. Don't disable it.
- **Errors:** retry transient errors with the SDK's retryer. Finish the batch instead of stopping at the first error. Show a summary of uploaded, skipped and failed files.
- **Pagination:** always use the SDK paginators. Never assume one page holds every object.
- **Destructive operations:** `--delete`, `rm --recursive` and `bucket rm --force` list what will be removed and ask for confirmation. Skip the prompt only with `--yes`. `--dry-run` never modifies anything. Use `DeleteMultipleObjects` in batches of up to 1000 keys.
- **Cancellation:** Ctrl-C in the CLI cancels the context (`signal.NotifyContext`), finishes cleanly, keeps checkpoints and prints a partial summary.

## Relationship to ossutil

- BucketOps **does not require** ossutil. All operations go through the Go SDK.
- The two tools share `~/.ossutilconfig`, so profiles made with `ossutil config` work in BucketOps without changes.
- `bucketops doctor` finds ossutil with `exec.LookPath("ossutil")` and reports its version from `ossutil version`. If it finds ossutil 1.x, it recommends upgrading to 2.0.
- If ossutil is missing, `doctor` prints install steps for the current OS and architecture. Never run the install automatically. Source: [install-ossutil](https://www.alibabacloud.com/help/en/oss/developer-reference/install-ossutil) and the [ossutil 2.0 overview](https://www.alibabacloud.com/help/en/oss/developer-reference/ossutil-overview/).
  - ossutil **2.0** is shipped as a zip file: `https://gosspublic.alicdn.com/ossutil/v2/<ver>/ossutil-<ver>-<os>-<arch>.zip`, for example `2.4.0` and `linux-amd64`. The steps are to unzip it, `chmod 755 ossutil` and `sudo mv ossutil /usr/local/bin/`. Keep the version in one constant so it's easy to update.
  - `curl https://gosspublic.alicdn.com/ossutil/install.sh | sudo bash` installs **1.x**, which is no longer updated. Don't recommend it.
- For reference, these are ossutil 2.0's own commands. BucketOps doesn't call them, but it can mirror their names where that helps users: `ls`, `cp -r`, `sync`, `cat`, `mb`, `rb`, `rm`, `stat`, `du`, `presign`, `restore`, `config --profile NAME`, plus `ossutil api <operation>` for low-level calls.

## Development commands

```bash
go mod tidy
go run ./cmd/bucketops                 # launch the TUI
go run ./cmd/bucketops --help          # CLI help
go build -o bin/bucketops ./cmd/bucketops
go test ./...
go test -race ./...
go vet ./... && golangci-lint run
gofmt -l . && goimports -w .
goreleaser release --snapshot --clean  # local cross-platform build
```

For TUI debugging, set `BUCKETOPS_DEBUG=1` to log to `~/.bucketops/debug.log` via `tea.LogToFile`. Stdout belongs to the TUI, so never print to it while the TUI is running.

## Coding guidelines

- **`internal/core` must not import cobra, bubbletea, bubbles or lipgloss.** It communicates only through return values, errors, channels and callbacks.
- The core depends on a small `OSSAPI` interface, not on `*oss.Client` directly, so tests can fake it.
- `keys` is pure and fully covered by table-driven tests, including Windows paths, trailing slashes, unicode, traversal attempts and every upload mode in the table above.
- Wrap errors with `fmt.Errorf("...: %w", err)`. Map SDK `*oss.ServiceError` codes (`NoSuchBucket`, `AccessDenied`, `InvalidAccessKeyId`, `SignatureDoesNotMatch` and the region/endpoint errors) to the `errs` sentinels in one place. The UIs use `errors.Is` to choose a friendly message.
- Pass `context.Context` as the first parameter. Never store contexts in structs.
- CLI exit codes: `0` for success, `1` for partial failure, `2` for usage or config errors.
- Test every TUI screen with teatest: navigation, the profile switch, both upload modes through the preview step, and cat preview.
- Run `go test -race` in CI, because transfers are concurrent.
