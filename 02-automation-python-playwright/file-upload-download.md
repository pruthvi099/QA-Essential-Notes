# File Upload & Download (Python & TypeScript)

## What It Is

File upload and download are two distinct browser interactions Playwright handles without needing OS-level automation (no need to control a native file picker dialog): uploads via `set_input_files()` / `setInputFiles()`, and downloads by intercepting the browser's download event via `expect_download()` / `waitForEvent('download')`.

## Why It Matters

- File upload/download flows (resumes, receipts, exported reports, profile pictures) are common in real applications, and naive automation attempts often try to interact with the native OS file picker dialog — which Playwright deliberately avoids needing, since OS-level dialogs aren't part of the browser's DOM and are notoriously unreliable to automate directly.
- Download testing requires explicitly waiting for the download event, since a click that triggers a download doesn't block/wait the way a normal navigation does — missing this is a common cause of tests that "pass" without actually verifying the download happened.
- This is a practical, frequently-needed skill in real QA automation work — interviewers use it to check whether a candidate has handled non-trivial, real-world scenarios beyond basic click/fill flows.

## How It Works

**Upload** — `set_input_files()` / `setInputFiles()` sets a file (or multiple files) directly on an `<input type="file">` element, bypassing the OS dialog entirely. Works even if the input is visually hidden (a common pattern where a styled button triggers a hidden native input).

**Download** — since clicking a download link doesn't pause execution while the file downloads, you must wrap the triggering action in an explicit wait for the `download` event, which gives you a `Download` object with methods to save the file, check its suggested filename, and get its path.

## Example

**Python — file upload:**
```python
from playwright.sync_api import Page

def test_upload_resume(page: Page):
    page.goto("https://jobs.example.com/apply")

    # Works directly on the <input type="file">, even if visually hidden
    page.set_input_files("input#resume-upload", "test-files/sample-resume.pdf")

    # Multiple files
    page.set_input_files("input#attachments", [
        "test-files/cover-letter.pdf",
        "test-files/portfolio.pdf",
    ])

    page.get_by_role("button", name="Submit Application").click()
    assert page.get_by_text("Application submitted").is_visible()
```

**Python — file download:**
```python
def test_download_invoice(page: Page):
    page.goto("https://shop.example.com/account/orders/123")

    with page.expect_download() as download_info:
        page.get_by_role("button", name="Download Invoice").click()

    download = download_info.value
    assert download.suggested_filename == "invoice-123.pdf"

    # Save it somewhere to inspect/verify content if needed
    download.save_as(f"downloads/{download.suggested_filename}")
```

**TypeScript — equivalent patterns:**
```typescript
import { test, expect } from '@playwright/test';

test('upload resume', async ({ page }) => {
  await page.goto('https://jobs.example.com/apply');

  await page.setInputFiles('input#resume-upload', 'test-files/sample-resume.pdf');

  await page.setInputFiles('input#attachments', [
    'test-files/cover-letter.pdf',
    'test-files/portfolio.pdf',
  ]);

  await page.getByRole('button', { name: 'Submit Application' }).click();
  await expect(page.getByText('Application submitted')).toBeVisible();
});

test('download invoice', async ({ page }) => {
  await page.goto('https://shop.example.com/account/orders/123');

  const downloadPromise = page.waitForEvent('download');
  await page.getByRole('button', { name: 'Download Invoice' }).click();
  const download = await downloadPromise;

  expect(download.suggestedFilename()).toBe('invoice-123.pdf');
  await download.saveAs(`downloads/${download.suggestedFilename()}`);
});
```

## Production Considerations

- Store test fixture files (sample PDFs, images) in a dedicated `test-files/` directory within the repo, kept small — large binary fixtures bloat repo size unnecessarily over time.
- For download tests, verify more than just the filename when it matters — reading and asserting on the downloaded file's actual content (e.g., parsing a downloaded CSV/PDF) gives much stronger confidence than filename-only checks, though this adds complexity and should be reserved for genuinely high-risk download flows.
- Clean up downloaded files after tests (or download to a temp directory that's wiped between runs) to avoid accumulating artifacts across CI runs.

## Common Pitfalls

- Trying to automate the native OS file picker dialog instead of using `set_input_files()`/`setInputFiles()` directly on the input element — this is unnecessary and unreliable; Playwright's approach bypasses the dialog entirely.
- Clicking a download trigger without first setting up the `expect_download()` / `waitForEvent('download')` wait — the test may "pass" without ever actually verifying a download occurred, since nothing failed, it just didn't check.
- Assuming `set_input_files()` requires the file input to be visible — it works on hidden inputs too, which is the common real-world case (a styled button + hidden native input).
- Not verifying downloaded content for high-risk download flows (e.g., a financial report) — filename-only checks can pass even if the file's actual content is wrong or corrupted.

## Interview Notes

- Be ready to explain why Playwright doesn't need to automate the native OS file picker dialog for uploads — a common question testing understanding of `set_input_files()`'s actual mechanism.
- Understand why download testing requires an explicit event wait, unlike most other Playwright interactions that rely on auto-waiting alone.
- Be able to describe how you'd verify a downloaded file's content, not just its existence/filename, for a high-risk download scenario.

## References

- [Playwright — File Uploads (Python)](https://playwright.dev/python/docs/input#upload-files)
- [Playwright — File Downloads (Node.js)](https://playwright.dev/docs/downloads)