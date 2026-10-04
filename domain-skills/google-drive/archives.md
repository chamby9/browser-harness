# Google Drive archive uploads and verification

## Navigation and file identity

- Folder URLs use `https://drive.google.com/drive/u/0/folders/<folder-id>`; file links use `https://drive.google.com/file/d/<file-id>/view`.
- Open a dedicated tab. A folder view is a SPA: wait after clicks and inspect the screenshot before assuming a menu or new row is ready.
- Rows and some descendants expose `data-id`. After visually confirming uploaded rows, collect IDs and visible names from these attributes; deduplicate repeated IDs. Do not treat a local Drive for desktop placeholder as evidence of a remote upload.

## Uploads

- Use the visible New → File upload menu. The hidden `input[type=file]` may mount only after the menu action and disappear after submission.
- Before clicking File upload, enable `Page.setInterceptFileChooserDialog`. Use `upload_file("input[type=file]", absolute_path_or_list)` when the input appears, then disable interception. This avoids leaving a native file picker open.
- The input supports multiple files. The upload panel displays progress, failures, and completed uploads. Confirm the completed state and folder rows independently.

## Downloads and integrity

- Configure `Browser.setDownloadBehavior` with a task-specific download directory and `eventsEnabled=True`; restore `behavior="default"` when finished.
- For a contiguous batch, click its first visible row, wait for selection to settle, then Shift-click its last row with CDP mouse events (`modifiers=8`). Wait again before right-clicking a selected row. Without the intervening waits, a context click can collapse selection.
- Download from the context menu. Batch downloads prepare a ZIP; "Download ready" is not proof that bytes reached disk. Wait for a completed browser download and disappearance of the `.crdownload` suffix.
- For large individual files, Drive may show a virus-scan size warning. Continue only for a file the task authorizes and knows. The subsequent "ready" state can still require an additional browser download link activation; inspect the current page/native browser state rather than repeatedly requesting downloads.
- A cloud archive is verified by reading the independent downloaded file (or ZIP member), matching byte length and SHA-256 to the original. Read ZIP members as streams to avoid another full extraction on a space-constrained disk.
- A browser upload completion check alone is insufficient justification for deleting unique local source data. Verify archive contents against current source, preserve recovery instructions, and check for active writers before removing originals.
- Filter download events to progress/completion metadata. Do not log signed download URLs, cookies, or unrelated network traffic.
