# OctSum CLI contract

## Installation and host setup

Install the OctSum desktop App, then Settings → Install CLI. Reopen the terminal/Agent after installation to pick up PATH changes. No development runtime is needed. Locate with `command -v octsum` on macOS; fallback is `$HOME/.local/bin/octsum`. On Windows use the host's **PowerShell** command tool, `Get-Command octsum -ErrorAction SilentlyContinue`, then `$HOME/.local/bin/octsum.cmd`. Invoke literal paths with the call operator, not by concatenating shell code:

```powershell
$cli = Join-Path $HOME '.local/bin/octsum.cmd'
& $cli -v
& $cli workspace list --json
& $cli submit 'https://www.bilibili.com/video/BV1bu411n7c7/?p=1' --action download --workspace 'C:\视频资料\我的 空间' --request-id 'REPLACE_WITH_UUID' --json
$LASTEXITCODE
```

Use PowerShell single-quoted literal URL/path arguments (double embedded single quotes). Do not send PowerShell syntax to cmd.exe or a POSIX shell. The installed `.cmd` launcher handles UTF-8. Do not use `Invoke-Expression`, `cmd /c` string construction, or shell expansion of video text. A host limited to a sandbox/cloud filesystem may not see the local launcher or App data; request its documented local execution permission. Do not alter global security settings. On macOS quote an absolute fallback as `"$HOME/.local/bin/octsum"`.

For WorkBuddy Mac, import `dist/skills/octsum-skill-<version>.zip` using Skills → Add Skill → local upload, enable it and select an Agent working folder. Check the loaded name/content matches this package and disable any older duplicate. Test explicit `/octsum-skill` invocation if offered by the host, then a Chinese request such as “用章鱼总结下载这个视频，先让我选择总结空间”。 Login, folder access and tool approvals belong to the user. Official instructions: [WorkBuddy Skills](https://www.workbuddy.ai/docs/workbuddy/From-Beginner-to-Expert-Guide/Function-Description/Skills).

WorkBuddy may ask for confirmation when SQLite cleans up `db.sqlite-journal` during a CLI transaction. If the intercepted command is the expected OctSum operation, ask the user to approve that specific operation through the host. Do not manually remove the journal or database, disable the deletion rule, or grant broad permanent access. Record this as a host permission gate, not a CLI/task failure; recover an uncertain submit with the original request ID.

## Commands

Examples use placeholders; replace them from actual responses. Never execute placeholders. Always keep the selected CLI path and workspace consistent.

```sh
octsum -v
octsum doctor --json
octsum workspace list --json
octsum submit 'https://www.bilibili.com/video/BV1bu411n7c7/?p=1' --action download --workspace '/absolute/我的 空间' --request-id 'REQUEST_UUID' --dry-run --json
# An explicit summary request authorizes the normal configured processing costs.
# Give a brief cost notice, preflight, then remove --dry-run to submit:
octsum submit 'https://www.bilibili.com/video/BV1bu411n7c7/?p=1' --action summarize --transcription remote --allow-paid --workspace '/absolute/我的 空间' --request-id 'REQUEST_UUID' --dry-run --json
octsum task result TASK_ID --workspace '/absolute/我的 空间' --json
octsum task wait TASK_ID --workspace '/absolute/我的 空间' --timeout 30 --json
octsum transcript show 'relative/video.mp4' --workspace '/absolute/我的 空间' --offset 0 --limit 100 --json
octsum summary show 'relative/video.mp4' --workspace '/absolute/我的 空间' --json
octsum summary export 'relative/video.mp4' --workspace '/absolute/我的 空间' --format md --output '/absolute/agent-folder/result-UUID.md' --json
octsum transcript export 'relative/video.mp4' --workspace '/absolute/我的 空间' --format srt --output '/absolute/agent-folder/transcript-UUID.srt' --json
octsum open --task TASK_ID --workspace '/absolute/我的 空间' --json
# Only when requested and supported:
octsum task retry TASK_ID --workspace '/absolute/我的 空间' --no-wait --json
```

`submit` supports Bilibili/YouTube single-video URLs and the existing Xiaohongshu explore/discovery links, including Bilibili `p`. It does not expand playlists or accept arbitrary media/CDN URLs. Actions mean download only, download + transcription, or download + transcription + summary (including configured markings). `auto` uses the saved global transcription mode and defaults to remote when none exists. Local models are checked only when inference is required after shared-transcript reuse. No vector model is required. `--dry-run` validates without creating work; an already submitted request ID may return its original task. Do not add `--force`, which submit does not support.

## Intent and necessary questions

The user's explicit processing request is the operation authorization for its normal configured costs. The CLI still requires `--allow-paid` for paid actions; the Agent supplies it after a short informational cost notice, without asking the user to approve the same action again. This does not authorize unrequested extra work, changing an explicit/saved local mode to remote, exceeding user restrictions, or repeating uncertain paid requests. Never treat “free only” as paid authorization. Download-only needs no paid flag.

| Situation                                                     | Behavior                                                                                                                                                               |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| “下载并总结”, chosen space and saved transcription preference | Keep `action=summarize`; notify briefly, preflight with `auto` and `--allow-paid`, then submit. Do not present an action/mode/fee questionnaire.                       |
| Same request, two spaces and no previous user choice          | Ask only where to save. Once chosen, continue the requested summary; do not offer download-only instead.                                                               |
| `auto` preflight returns `TRANSCRIPTION_REQUIRED`             | Try local if the reported local model is available. If none is available, ask only whether to install local models or use remote transcription; keep the summary goal. |
| Explicit/saved local mode fails with `MODEL_REQUIRED`         | Explain the missing local prerequisite. Remote substitution needs a new choice, not automatic fallback.                                                                |
| “下载并总结，不要扣费”                                        | Resolve the fee restriction before paid work; do not add `--allow-paid` just because summary was requested.                                                            |

Reuse a genuine user-selected space from the conversation or host memory, validating it against the current list. With one registered space select it directly; with multiple unresolved choices ask once. Do not turn an acceptance-test receipt into a normal default, silently use the App's last space, or persist settings on the user's behalf without a request.

## JSON and exits

Parse stdout as a single JSON value; diagnostic stderr can be nonempty even on success. Never merge the streams. New versioned interfaces are `submit`, `task result`, `open`, and **task wait with --timeout**. Success has `schemaVersion: 1`, `ok: true`, `data`; failure has `ok: false`, `error: {code, message, retryable}`. Timeout/interrupt may additionally retain `data.taskId` and `data.waitOutcome`. `retryable: false` is not the same as task-level `canRetry`.

Submission data includes `taskId` (string) and reuse/submission flags. Dry-run data includes `workspaceId`, normalized `url`, `action`, resolved `transcription`, `mayCharge`, `submitted: false`. Result data includes `taskId`, `status`, `completed`, `failed`, `stage`, `progress` (0–1 or null), `error`, `result`, `canRetry`, `stages`, `progressDetail`, and `resultAvailable` when a media path is known. Read `result.relativePath` only after checking terminal success and availability. A parent may expose download results while later work is running.

| Exit | Versioned interface meaning                                                  |
| ---- | ---------------------------------------------------------------------------- |
| 0    | Command succeeded; still inspect task completion                             |
| 1    | Execution/query error, or queried task failed/aborted                        |
| 2    | Invalid arguments/URL or request conflict                                    |
| 3    | Missing workspace, activation, model, saved preference or paid authorization |
| 4    | Wait timeout; task continues                                                 |
| 130  | Interrupted wait; task continues                                             |

Stable codes include `INVALID_ARGUMENT`, `INVALID_URL`, `REQUEST_CONFLICT`, `WORKSPACE_REQUIRED`, `MODEL_REQUIRED`, `TRANSCRIPTION_REQUIRED`, `ACTIVATION_REQUIRED`, `PAID_AUTHORIZATION_REQUIRED`, `TASK_NOT_FOUND`, `TASK_RESULT_MISSING`, `EXECUTION_FAILED`, `TIMEOUT`, `INTERRUPTED`. Handle codes, not localized text. A result query can have `ok: true` and `failed: true` with exit 1.

**Legacy commands stay unwrapped:** `doctor`, `workspace list`, `task list/info/retry/cancel`, `transcript show/export`, `summary show/export`. Their errors generally use `{"ok":false,"error":"localized message"}` and exit 1; do not apply the new exit mapping/envelope blindly.

- `workspace list`: array of `{id, name, localPath, ...}` (camelCase). An empty array means create/select a space in the App, not guess the current folder.
- `doctor`: object with `cliVersion`, `protocolVersion`, `downloads.ytDlp`, `downloads.bundledQuickJs`, `ffmpeg` (`{"Ok":"path"}` or `{"Err":"reason"}`), `resources.fsts`, `localTranscriptionModel.available`, `remoteConfigured`, `failed`. It checks resources/config presence, **not** live activation or balance. `failed` counts missing bundled resources; exit 1 can coexist with parseable diagnostics. Do not dump full diagnostic paths/config into a public report.
- `summary show`: actual summary object or null, without `data`; `transcript show`: transcript with `segments` and `pagination: {offset, limit, total}` or null. Transcript pages default 100, max 500; task list also paginates. Read output before deciding which fields contain useful content.
- Show results over 128 KiB require export. Export with `--output` returns `{path, format}` and refuses overwrite. Use a new destination; verify it exists and contains the requested result. Never overwrite user files.

## Receipts and recovery

Use the host's own file-write/read tools, no helper runtime. Before submitting, save this shape under the Agent work folder `.octsum-skill/runs/<request-id>.json`:

```json
{
  "requestId": "UUID",
  "workspace": "/absolute/OctSum-space",
  "url": "https://www.bilibili.com/video/BV1bu411n7c7/?p=1",
  "action": "download",
  "transcription": "auto",
  "allowPaid": false,
  "taskId": null,
  "state": "prepared"
}
```

After submit set the returned `taskId` and `state: "submitted"`; optionally record timestamps, last status and export path. If the receipt already exists, read it before writing; never replace another operation. Receipt writes precede side effects. Keep IDs as strings. On interruption resume by task ID; if unknown, resend **the same** recorded parameters/request ID. Never change defaults to repair a request conflict. A deleted result stays historically completed but is not usable; a fresh download needs a new user operation/ID. Keep failed evidence, excluding secrets. A new local task ID, duplicate filename, short elapsed time or previously empty local transcript does not establish whether the service reused its remote cache. Require an explicit service signal to classify cache provenance; otherwise record unknown, even when an upstream error includes a task ID.

## Tracking budget

A CLI wait timeout is a polling boundary, not the workflow deadline. Keep calling `task wait <id> --timeout 30 --json` while the task is active and within its overall budget, shortening the final call to the remaining seconds. Allow startup overhead in the host's external command timeout, for example 45 seconds. There is no three-call/90-second per-turn cap; do not force the user to send “continue” for a normally running task.

Add only the stages still needed when tracking starts. There is **no minimum overall budget**:

| Stage                         | Tracking allowance            |
| ----------------------------- | ----------------------------- |
| Download                      | 300 seconds                   |
| Transcription                 | `ceil(1.2 × durationSeconds)` |
| Summary                       | 120 seconds                   |
| Configured follow-up markings | 180 seconds total             |

The transcription factor comes from the user's planning estimate. The other allowances are provisional waiting limits, not measured service timings. Do not reserve summary or marking time for an action that will not run them; omit already completed/reused stages when their completion is known. For example, a one-minute video needs 6 minutes 12 seconds for download + transcription, 8 minutes 12 seconds with summary, or 11 minutes 12 seconds if markings are also enabled. Download-only gets five minutes, not a ten-minute minimum.

Obtain duration from existing trustworthy metadata or, after the media path exists, `octsum file info '<relativePath>' --workspace '<absolute-space>' --json` → `index.durationMs / 1000`. Ignore missing/nonpositive values; do not add a second network probe just to get duration. Unknown transcription duration provisionally contributes 900 seconds, independently of the other stages. Record the initial stage allowance breakdown in the receipt. Once reliable duration is known, replace only this provisional transcription allowance and recompute the deadline from the original tracking start, not from now. Never add allowances again on each poll, stage change or resumed turn. If a user gives a shorter tracking deadline, honor it. Queues and service timings can exceed these allowances; do not promise completion within the budget or display estimates as real progress.

Persist `trackingStartedAt` and `trackingDeadlineAt` in the existing request receipt. On resumption use the saved deadline; for older receipts without these fields prefer the recorded submission time or actual task timestamp, otherwise record the first tracking start once. Do not reset the entire budget on every turn. If the saved deadline has expired, query current status once and report it; further waiting requires the user to request an extension. It never requires a fresh submission.

Provide concise progress updates about once a minute or on stage changes, without requiring a reply. Continue automatically after exit 4/TIMEOUT. Exit 130/user interruption stops waiting. Genuine host execution/permission limits may prevent staying in one turn; report that limit and retain the original ID instead of claiming the task failed. At the overall deadline query once more, then report the unfinished task with stage and ID; do not cancel it. A failed/aborted task or deleted/empty result remains a failure/unavailable result, even when downloading completed. `open` acknowledges dispatch only; the App might require its normal OS launch permissions.
