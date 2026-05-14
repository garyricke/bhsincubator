# Piron Scholarship Comments — Google Sheets backend setup

The comments section on `piron-scholarship.html` reads from and writes to a Google Sheet via a Google Apps Script Web App. Visitors post a comment → it's appended to the sheet → it appears on the wall. To moderate, you delete a row in the sheet.

This is a **one-time setup**. After you finish, the page just works.

## 1. Create the Google Sheet

1. Go to <https://sheets.new>
2. Name the file something like **"Piron Scholarship Comments"**
3. In row 1 (the header row), put:

   | A    | B            | C       | D         |
   | ---- | ------------ | ------- | --------- |
   | name | relationship | message | timestamp |

   Type those four labels into A1, B1, C1, D1 exactly (lowercase).

## 2. Open the Apps Script editor

1. In the sheet, click **Extensions → Apps Script**
2. A new tab opens with an editor and an empty `Code.gs` file
3. Delete anything in there and paste the entire block below:

```javascript
// Piron Scholarship comments — Google Apps Script backend
// Bound to the "Piron Scholarship Comments" sheet
// Headers in row 1: name | relationship | message | timestamp

const MAX_NAME = 60;
const MAX_REL = 60;
const MAX_MSG = 1000;

function doGet(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const values = sheet.getDataRange().getValues();
  values.shift(); // drop header row

  const comments = values
    .filter(row => row[2] && String(row[2]).trim().length > 0)
    .map(row => ({
      name: String(row[0] || '').trim(),
      relationship: String(row[1] || '').trim(),
      message: String(row[2] || '').trim(),
      timestamp: row[3] instanceof Date ? row[3].toISOString() : String(row[3] || '')
    }))
    .reverse(); // newest first

  return ContentService
    .createTextOutput(JSON.stringify({ status: 'ok', comments }))
    .setMimeType(ContentService.MimeType.JSON);
}

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);

    // Honeypot — bots fill this hidden field; humans don't
    if (data.website && String(data.website).trim().length > 0) {
      return jsonOut({ status: 'ok' }); // silently drop
    }

    const name = String(data.name || '').trim().slice(0, MAX_NAME);
    const relationship = String(data.relationship || '').trim().slice(0, MAX_REL);
    const message = String(data.message || '').trim().slice(0, MAX_MSG);

    if (!message) {
      return jsonOut({ status: 'error', error: 'Message is required.' });
    }

    const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    sheet.appendRow([name, relationship, message, new Date()]);

    return jsonOut({ status: 'ok' });
  } catch (err) {
    return jsonOut({ status: 'error', error: String(err) });
  }
}

function jsonOut(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

4. Click the **disk icon** (Save) — name the project "Piron Scholarship Comments API"

## 3. Deploy as a Web App

1. Top-right of the editor: **Deploy → New deployment**
2. Click the gear icon next to "Select type" → choose **Web app**
3. Settings:
   - **Description:** "Piron Scholarship comments v1"
   - **Execute as:** `Me (your email)`
   - **Who has access:** **Anyone** ← critical, NOT "Anyone with Google account"
4. Click **Deploy**
5. The first time, Google asks you to authorize: click **Authorize access** → pick your account → click **Advanced** if you see a "Google hasn't verified this app" screen → click **Go to Piron Scholarship Comments API (unsafe)** → **Allow**
6. After deployment, copy the **Web app URL**. It looks like:
   `https://script.google.com/macros/s/AKfycb.../exec`

## 4. Paste the URL into the page

Open `piron-scholarship.html`, find this line near the top of the `<script>` block:

```javascript
const COMMENTS_API_URL = 'PASTE_YOUR_APPS_SCRIPT_URL_HERE';
```

Replace the placeholder with the URL you just copied. Save, commit, push.

## 5. Test

1. Open the page
2. Scroll to the comments section
3. Submit a test comment ("Test from Gary", "Test relationship", "Hello world")
4. It should appear on the wall within ~2 seconds
5. Open the sheet — you should see the row appended
6. Refresh the page — the comment should still be there

## Moderating

- Delete a row in the sheet → it disappears from the wall on next page load
- Edit a row in the sheet → updates the wall on next page load
- No re-deployment needed — Apps Script reads the live sheet every time

## If you redeploy / change the script

If you edit `Code.gs` later, you need to **Deploy → Manage deployments → pencil icon → Version: New version → Deploy** to push the changes. Reusing the same deployment keeps the URL stable so the page doesn't need updating.

## Spam protection in place

- Honeypot hidden field — bots fill it, humans don't
- 1,000-character message limit (server-side)
- 60-character name/relationship limit (server-side)
- Client-side: 30-second rate limit per browser via localStorage

If something does slip through, just delete the row in the sheet.
