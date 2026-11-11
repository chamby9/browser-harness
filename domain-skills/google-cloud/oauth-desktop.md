# Google Auth Platform: private Desktop OAuth clients

## Routes and page state

- The current console uses `https://console.cloud.google.com/auth/branding?project=<project-id>`, `/auth/audience`, `/auth/scopes` and `/auth/clients`.
- Open a dedicated task tab and select its target ID before every invocation. Another task or user can change the daemon's selected tab between calls.
- The console is a SPA. `wait_for_load()` can return before a route or side panel has settled, or wait on unrelated background loading. Verify the expected heading/field and final screenshot. An immediately captured panel can still be sliding into position.
- CDP mouse input uses CSS coordinates; screenshots can use a different device pixel ratio. Read the actual image dimensions and `devicePixelRatio` before converting. Do not reuse old geometry after a validation message disappears or a panel changes width.

## Durable personal-use authorisation

- A personal Google account requires External audience. Desktop identifies the OAuth client type, not the audience.
- Branding initially allows app name, support email and developer contact to be saved without public URLs. Audience may nevertheless disable **Publish app** with "complete your configuration on the Branding page". Repeatedly saving the same three fields does not resolve that state.
- Current [Google branding requirements](https://support.google.com/cloud/answer/15549049) require public home/privacy links for external production apps. Use accurate pages the owner has authorised publishing. Add the exact domain requested by the console under Authorized domains, save, then inspect Audience again. Do not invent placeholder URLs or publish private data to fill the fields.
- App publishing status is separate from brand verification. Personal-use clients can be exempt from verification. A logo and sensitive/restricted scopes can add verification requirements. Choose only the scope actually needed; `drive.file` is suitable for an archive that creates and manages its own Drive objects.
- Testing-mode Drive authorisations expire after seven days. Production status removes that test-mode limit. After changing status, obtain fresh OAuth credentials through the normal browser flow; do not assume an old Testing token became durable. See [Audience](https://support.google.com/cloud/answer/15549945) and [verification exemptions](https://support.google.com/cloud/answer/13464323).
- A test account must actually be saved in Audience before Testing sign-in works. A chip in the Add users panel is not the saved list. Confirm the closed panel and resulting user row. After saving, retry the normal OAuth flow; the old access-denied page does not update itself.

## Credentials and consent

- Desktop client creation may display the client secret in its success dialog. Do not screenshot or extract the whole dialog. Locate only the download/close controls and download JSON directly into a private directory. Restore default browser download behaviour afterwards.
- If the task requires a mandatory policy agreement, obtain the applicable user approval before accepting it. An ordinary configuration task is not evidence of agreement with a newly presented policy.
- A local OAuth callback URL can include a state value. Reopening only the bare `/auth` path can produce "State did not match". Retry the exact URL issued by the active local process; keep the URL and credential-bearing logs private.
- Successful authorisation normally returns a local Success page while the CLI may still await a Shared Drive question. Check the original CLI process and finish its remaining prompt. Browser success alone does not establish that credentials were saved or that a real Drive operation works.
