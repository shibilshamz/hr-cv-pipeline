# Setup

Getting this running on your own n8n instance. Budget about 20 minutes, most of it Google OAuth.

## Prerequisites

- An n8n instance (self-hosted or cloud) — see [Self-hosting n8n](#self-hosting-n8n) below
- A Google account with Drive and Sheets
- An Anthropic API key from [console.anthropic.com](https://console.anthropic.com)

---

## 1. Create the three Drive folders

In Google Drive, create:

```
CV Pipeline/
├── Inbox      ← recruiters drop CVs here
├── Done       ← processed successfully
└── Errors     ← couldn't be read; needs a human
```

Open each folder and copy its ID from the URL:

```
https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOpQrStUvWxYz
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^ this part
```

Keep all three IDs handy.

## 2. Create the spreadsheet

Create a new Google Sheet. Paste the header row from [`docs/sheet-template.csv`](docs/sheet-template.csv) into row 1 — all 16 columns, exact spelling and capitalisation, since the workflow maps by column name:

```
Full Name | Contact Number | Nationality | Email Id | Current Location | Visa Status |
Last Company | Last Job Title | Relevant Years of Experience | GCC Experience |
Current Salary | Expected Salary | Notice Period | Reason for Leaving |
Source File | Processed At
```

Copy the spreadsheet ID from its URL:

```
https://docs.google.com/spreadsheets/d/1XyZ.../edit#gid=0
                                       ^^^^^^ this part
```

The workflow targets the **first tab** (`gid=0`). If you want a different tab, update the `sheetName` value in both Sheets nodes after import.

## 3. Add credentials in n8n

**Settings → Credentials → Add credential**, three times:

| Credential type | Used by | Notes |
|---|---|---|
| Google Drive OAuth2 API | Watch Inbox, Download CV, Move to Done, Move to Errors | Needs Drive read + write (files get moved) |
| Google Sheets OAuth2 API | Check Duplicate, Append row in sheet | |
| Header Auth *(recommended)* | Claude Extract | Name: `x-api-key`, Value: your Anthropic key |

Google OAuth setup is the standard n8n flow — n8n's [Google credential docs](https://docs.n8n.io/integrations/builtin/credentials/google/) walk through the Cloud Console project and redirect URI.

**On the Anthropic key:** the published workflow has `YOUR_ANTHROPIC_API_KEY` as a literal header value in the `Claude Extract` node. Replacing that string works, but the key is then stored in the workflow itself and travels with every export and backup. Prefer creating a **Header Auth** credential and switching the node's authentication to use it — the key stays in n8n's encrypted credential store instead.

## 4. Import the workflow

**Workflows → Import from File** → `workflow/cv-data-extractor.json`.

It imports **inactive** by design, so it doesn't start polling before you've pointed it at your own folders.

## 5. Replace the six placeholders

Open each node and swap in your own IDs:

| Placeholder | Node(s) | Replace with |
|---|---|---|
| `YOUR_INBOX_FOLDER_ID` | Watch Inbox | Inbox folder ID |
| `YOUR_DONE_FOLDER_ID` | Move to Done | Done folder ID |
| `YOUR_ERRORS_FOLDER_ID` | Move to Errors | Errors folder ID |
| `YOUR_SPREADSHEET_ID` | Check Duplicate, Append row in sheet | Spreadsheet ID |
| `YOUR_DRIVE_CREDENTIAL_ID` | 4 Drive nodes | Pick your Drive credential from the dropdown |
| `YOUR_SHEETS_CREDENTIAL_ID` | 2 Sheets nodes | Pick your Sheets credential from the dropdown |
| `YOUR_ANTHROPIC_API_KEY` | Claude Extract | Your key, or switch to Header Auth (see above) |

Credential placeholders resolve themselves once you select the credential in the node UI — you don't edit those IDs by hand.

## 6. Test, then activate

Drop one PDF CV into Inbox and click **Execute Workflow** to run it manually. You should see:

- a new row in the sheet
- the PDF moved from Inbox to Done

Then toggle **Active**. It polls every minute from there.

---

## Self-hosting n8n

The original runs in Docker on an Ubuntu VPS, with the container bound to localhost and a reverse proxy in front of it:

```bash
docker run -d --restart unless-stopped \
  --name n8n \
  -p 127.0.0.1:5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e GENERIC_TIMEZONE="Asia/Dubai" \
  -e TZ="Asia/Dubai" \
  n8nio/n8n
```

Binding to `127.0.0.1` keeps port 5678 itself off the internet. You then reach the editor one of two ways.

**Private — SSH tunnel, nothing exposed:**

```bash
ssh -L 5678:127.0.0.1:5678 user@your-vps
```

and open `http://localhost:5678`.

**Public — reverse proxy with TLS.** A minimal Caddyfile:

```caddyfile
n8n.example.com {
    reverse_proxy localhost:5678
}
```

⚠️ **A reverse proxy puts your n8n login page on the public internet**, and the localhost binding no longer protects it — the proxy runs on the same host. Anything stored in a workflow (API keys pasted into HTTP nodes, spreadsheet IDs) is one login away from anyone who finds the URL. If you go this route:

- Use a strong, unique password and **turn on 2FA** in n8n
- Keep credentials in n8n's credential store, not inline in nodes
- Consider `basic_auth` in the proxy, or an IP allowlist, as a second layer
- Don't rely on an obscure hostname — wildcard-DNS names like `<ip>.sslip.io` are derivable from the IP and show up in your TLS certificate

Never publish port 5678 directly.

---

## Troubleshooting

**Nothing happens when I drop a file.** The Drive trigger polls on a schedule — give it a minute. It also only fires on `fileCreated`; moving an existing file *out of* another Drive folder may not register as a create. Re-upload it instead.

**Every CV lands in Errors.** `Extract Text` can't read image-only PDFs. Scanned CVs need OCR before this pipeline — that's a separate node ahead of step 3.

**Phone numbers show `#ERROR!`.** The apostrophe prefix in `Parse Response` was removed or the column is formatted as a number. Set the Contact Number column to Plain Text formatting.

**Rows are missing but the CVs moved to Done.** They were flagged as duplicates. Dedup is by email — a CV with no email address on it is always treated as a duplicate and skipped, on purpose.

**`Check Duplicate` is slow.** It reads the whole sheet on every run. Past a few thousand rows, add a `filtersUI` filter or archive old rows to a second tab.
