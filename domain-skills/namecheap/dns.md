# Namecheap DNS host records

## Navigation and scope

- The account panel is `https://ap.www.namecheap.com/`. Open a dedicated tab and use the signed-in account; stop at a login wall.
- Domain List → Manage opens `/domains/domaincontrolpanel/<domain>/domain`. Advanced DNS is the next tab. Confirm the exact domain heading before editing.
- BasicDNS/PremiumDNS/FreeDNS exposes Host Records. Keep nameservers and Mail Settings unchanged when only connecting a website.

## Inline editing

- Clicking a record's value opens the whole row for editing. The green check saves that row; the orange cross cancels. Save All Changes appears while rows are being edited.
- A record's Type can also be changed inline. The dropdown opens scrolled to its current choice. Scroll within the dropdown to reveal earlier types such as A Record.
- Changing URL Redirect Record to A Record retains the old URL in the value field. Explicitly clear and replace it with the required IP before saving.
- Save is asynchronous. A screenshot immediately after clicking the green check may still show editing controls. Wait until the row exits editing, then reload and verify persistence.
- For select-all on macOS fields, raw `Input.dispatchKeyEvent` with `commands:["selectAll"]` works when an ordinary Cmd+A helper does not. Read the focused input value before saving to ensure replacement, not appended text.

## Connecting a hosting provider

- Obtain exact A/CNAME values from the target project's current provider settings. Avoid copying another project's DNS values or assuming old generic documentation is current.
- New domains commonly have an apex URL Redirect and a www CNAME to a parking page. Replace those conflicting website records while preserving unrelated records.
- Apex Host is `@`; a www Host is `www`. Use a DNS target, not an https URL, for CNAME values.
- Verify against an authoritative nameserver after saving. Recursive caches may retain previous values for the old TTL.
- A correct DNS record does not establish HTTPS readiness. Verify the certificate chain/hostname and actual redirects, then exercise the visitor journey on the new origin.
