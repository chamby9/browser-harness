# Uploads

When a site exposes a file upload, you usually need to drive a hidden `<input type="file">`. Don't try to interact with the native OS file picker — browser-harness can't reach it.

## The pattern

Set the file directly on the input via CDP `DOM.setFileInputFiles`:

```python
# 1. Tag the hidden input so we can find it via CSS selector
js('''
  const inp = document.querySelector('input[type="file"]');
  if (inp) inp.id = 'bh_file_target';
''')

# 2. Resolve nodeId via CDP and attach the file
doc = cdp('DOM.getDocument')
sel = cdp('DOM.querySelector', nodeId=doc['root']['nodeId'], selector='#bh_file_target')
cdp('DOM.setFileInputFiles', nodeId=sel['nodeId'], files=['/absolute/path/to/file.jpg'])
```

The path must be **absolute** and the file must exist on the same machine the daemon is running on. Sites usually pick up the change automatically because they're listening for input events on the file input — no extra dispatch needed.

## When the file input is wrapped in a button

Many UIs hide the input behind a custom-styled button. The button's click handler typically calls `input.click()` to open the native picker. You don't need to click the button at all — just find the hidden input directly (it lives in the DOM whether or not the button has been clicked) and drive it via CDP as above.

If the input only mounts on click (rare, but happens with React-controlled portals), click the button first, then immediately query for the input. The file picker will open natively, but you can ignore it — setting the input via CDP fires the right events without the picker ever being touched.

## Multi-file inputs

`DOM.setFileInputFiles` takes a list:

```python
cdp('DOM.setFileInputFiles', nodeId=sel['nodeId'], files=['/path/a.jpg', '/path/b.jpg'])
```

The site needs `<input multiple>` for this to work — a single-file input will accept only the first.

## Verify

After setting, read back to confirm:

```python
val = js('return document.querySelector("#bh_file_target").files.length')
```

If 0, the path was wrong or the input was replaced. Re-query and try again.

## Why not click → keypress?

The native file picker is an OS-level dialog. CDP does not control it. You'd need OS-level UI automation (osascript on macOS, AutoHotkey on Windows) which is platform-specific, fragile, and unnecessary when `DOM.setFileInputFiles` works on every Chromium browser via the same CDP path.
