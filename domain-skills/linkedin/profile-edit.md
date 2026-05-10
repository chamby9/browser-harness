# LinkedIn — Profile Edit

Write to a LinkedIn profile: headline, About, Experience descriptions, Skills (add / delete / reorder), and the profile photo. Companion to `profile-scrape.md`, which covers reads.

## Edit URLs are deterministic — use them, don't hunt for pencils

Each editable item has a stable URL. Scrape once via `aria-label` to capture the IDs, then navigate directly:

```
# Top of profile (name, headline, current position, location, industry, contact)
https://www.linkedin.com/in/<vanity>/edit/intro/

# Single experience role
https://www.linkedin.com/in/<vanity>/details/experience/edit/forms/<role-id>/

# Single skill
https://www.linkedin.com/in/<vanity>/details/skills/edit/forms/<skill-id>/

# Add a new skill
https://www.linkedin.com/in/<vanity>/skills/edit/forms/new/
```

To enumerate IDs, scrape the listing pages once:

```python
els = js('''
  const out = [];
  document.querySelectorAll('a').forEach(e => {
    const lbl = (e.getAttribute('aria-label') || '').trim();
    if (lbl.startsWith('Edit ')) {
      out.push({lbl, href: e.href || ''});
    }
  });
  return out;
''')
```

You'll get entries like `Edit Associate Manager | Solution Architecture at nCino, Inc.` mapped to a numeric form ID. Keep the dict around for the rest of the session.

### About has no direct edit URL

There's no `/edit/about/` route. To add an About section that doesn't yet exist, click **Add section → Core → Add about**. To edit an existing one, find the pencil/edit on the About card itself. This is one of the few sections that requires UI clicks rather than a direct URL.

## Headline is contenteditable, not `<input>` or `<textarea>`

The Headline field inside the Edit intro modal is a `<div contenteditable="true">`. Standard `Meta+a` selection is unreliable in browser-harness on macOS — the keypress doesn't always reach the field. Use the Selection API instead:

```python
js("""
  const ce = document.querySelector('p:has-text("Headline*") + * [contenteditable="true"]')
            || /* find via DOM walk from the 'Headline*' label */;
  ce.focus();
  const range = document.createRange();
  range.selectNodeContents(ce);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
  document.execCommand('delete', false, null);
""")
```

Then insert the new value via `execCommand('insertText', ...)` so React's input listeners fire:

```python
import json
js(f"document.execCommand('insertText', false, {json.dumps(new_headline)})")
```

`type_text()` after a manual click also works for fresh entries, but if you don't fully clear first you'll prepend rather than replace. Backspace-spamming hundreds of times is slow and unreliable; the Selection + execCommand pattern is the durable answer.

## Description fields in Experience modals are real `<textarea>`s

Different shape, different setter. Use the React-controlled native value setter so React's onChange fires:

```python
js(f'''
  const ta = document.querySelector('textarea[aria-label*="Description"]');
  ta.focus();
  const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
  setter.call(ta, {json.dumps(text)});
  ta.dispatchEvent(new Event('input', {{bubbles: true}}));
''')
```

The textarea is below the visible viewport on most screens. Run `ta.scrollIntoView({block: 'center'})` first if you also need to capture coordinates for any reason.

## About uses a contenteditable like Headline

Same pattern as Headline — Selection + `execCommand('delete')` to clear, `execCommand('insertText', ...)` to insert.

`\n` and `\n\n` in `insertText` produce paragraph breaks in the rendered About section. Do not double-Enter via `press_key('Enter')` to create paragraph breaks — the contenteditable inserts a fresh `<p>` per Enter and you end up with multiple blank lines per gap. Use `\n\n` inside one `insertText` call.

## Modal Save buttons live on the dialog

Each `<dialog>` element has its Save button at the bottom right. Find by inner text:

```python
save = js('''
  const dialog = document.querySelector('dialog');
  const btns = dialog?.querySelectorAll('button') || [];
  for (const b of btns) {
    if (b.innerText && b.innerText.trim() === 'Save') {
      const r = b.getBoundingClientRect();
      if (r.width > 0) return {x: Math.round(r.x + r.width/2), y: Math.round(r.y + r.height/2)};
    }
  }
  return null;
''')
```

Coordinates land around `(996, 915)` on a 1280×1323 viewport, but always re-query — modal heights vary by section. If the Save sits below the viewport, run `scrollIntoView` on the textarea/input first; that often pulls Save into view too.

## Post-save side prompts (the annoying ones)

After saving an Experience role you may get redirected through a "next-action" flow:

1. **Verify your employment** — LinkedIn offers an employment-verification badge via work email. Look for a **Skip** link (it's an `<a>`, not a `<button>`):
   ```python
   skip = js('''
     const els = document.querySelectorAll('a, button');
     for (const e of els) {
       if (e.innerText && e.innerText.trim() === 'Skip') {
         const r = e.getBoundingClientRect();
         if (r.width > 0) return {x: Math.round(r.x + r.width/2), y: Math.round(r.y + r.height/2)};
       }
     }
     return null;
   ''')
   ```
2. **People you may know** — appears at `/edit/forms/next-action/people-you-may-know/<companyId>/`. Just navigate away.

Both are skippable and not part of the Save itself; the underlying section data has already persisted by the time these appear.

## Skills

### Adding

Hit the new-skill URL, type the skill, and accept an autocomplete option:

```python
goto_url('https://www.linkedin.com/in/<vanity>/skills/edit/forms/new/')
# Click the input, type the skill, wait, then click the first '[role="option"]' result.
```

**Trap — verify the picked skill before saving.** LinkedIn's autocomplete is opinionated. `Software Engineering` resolves to `Engineering Management` (different skill). After clicking the option, read the input's actual value and confirm:

```python
chosen = js('return document.querySelector(\'input[placeholder*="Skill"]\').value')
```

If wrong, abort and try a different phrasing (`Software Engineering Practices`, `Software Development`, etc.).

The form also surfaces a "Show us where you used this skill" experience checklist. Tick the most relevant role (usually the current one) before save — this links the skill to a position and makes it appear in that role's skills tag list on the public profile.

### Deleting

Open the skill's edit URL. The modal has a `Delete skill` button at the bottom-left. Click it, then confirm in the secondary dialog (`Delete from profile? Delete X from all sections on your profile? This action cannot be undone.`) by clicking `Delete`.

### Reordering — the kebab on the details page

The Skills section on the public profile auto-orders by endorsements + recency. To force a specific top-3 order you have to use the **Reorder** modal, which is hidden behind the kebab on the details page:

```
https://www.linkedin.com/in/<vanity>/details/skills/
→ click the kebab next to the "Skills" heading (aria-label is unhelpful, button has the text "Resources" or just shows three dots)
→ click "Reorder"
```

The reorder modal lists ALL skills, including auto-imported ones from job-search behaviour you may not know exist. Drag handles are on the right edge of each row.

**There is no Save button.** Changes commit on dialog close. Just dismiss the dialog when you're done.

### Drag-to-reorder via raw CDP

`Input.dispatchMouseEvent` is the only reliable way; HTML5 drag-and-drop events don't fire correctly through CDP without the full mousedown → mousemove sequence:

```python
sx, sy = handle_x, source_y       # drag handle x is around right edge of dialog
tx, ty = handle_x, target_y

cdp('Input.dispatchMouseEvent', type='mousePressed', x=sx, y=sy, button='left', clickCount=1)
time.sleep(0.3)
steps = 20
for i in range(1, steps + 1):
    nx = sx + (tx - sx) * i / steps
    ny = sy + (ty - sy) * i / steps
    cdp('Input.dispatchMouseEvent', type='mouseMoved', x=int(nx), y=int(ny), button='left')
    time.sleep(0.05)
time.sleep(0.3)
cdp('Input.dispatchMouseEvent', type='mouseReleased', x=tx, y=ty, button='left')
```

Drop targets behave like "insert above this row." Drag-overshoot can land an item one slot off the intended position. After each drag, re-query the rendered order and adjust.

## Profile photo upload

The "Update" button in the photo modal triggers a hidden file input. Don't try to drive the native macOS file picker via UI automation — set the input directly via CDP:

```python
# Tag the hidden input
js('document.querySelector(\'input[type="file"][accept="image/*"]\').id = "bh_file_target"')

# Set the file
doc = cdp('DOM.getDocument')
sel = cdp('DOM.querySelector', nodeId=doc['root']['nodeId'], selector='#bh_file_target')
cdp('DOM.setFileInputFiles', nodeId=sel['nodeId'], files=['/absolute/path/to/photo.jpg'])
```

LinkedIn then renders an "Edit image" preview with Crop / Filter / Adjust tabs. The user may want to fine-tune; if not, click `Save changes` (yes, this Save says "Save changes" not "Save").

After save you'll briefly see a "Saving..." toast. Refreshing the profile shortly after confirms the new photo (look for a fresh image ID in the `media.licdn.com/dms/image/v2/<NEW_ID>/profile-displayphoto-...` URL).

### Source resolution

LinkedIn caches the displayed avatar at 400×400 even when the original is much larger. If you only have access to the cached version, that's the size you'll work with. For background-replace work, ask the user to pull the original full-resolution file from their device — it makes a visible difference at LinkedIn's various display sizes.

## Notify-network toggle

The Edit experience modal has a "Notify network" toggle at the top. **Default is OFF, leave it off** unless the user explicitly wants to broadcast each tweak. Multiple saves with this on will spam the user's network with "started a new role" updates per edit — embarrassing during an iterative profile rewrite.

## What you do NOT touch without explicit permission

- The profile photo (until colour / framing is confirmed by the user).
- "Open to work" badge state — has implications for the user's discoverability and current employment optics.
- Connections — never accept, decline, or send invitations as part of edit work.
- Activity / posts — never delete posts, never post on the user's behalf.
- Privacy settings.
- Email / contact info.

## Auth

Same as the scrape side — the harness attaches to the user's existing Chrome session, so log-in state is theirs. If you hit a login wall mid-edit, stop and ask. Saves happen against the live, real account.
