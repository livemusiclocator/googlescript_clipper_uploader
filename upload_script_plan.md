# Google Apps Script Plan: Upload Gigs from Sheet to LML Admin

## Goal

Create a Google Apps Script associated with a Google Sheet to automate the process of uploading gig data to the LML admin endpoint (`/admin/uploads`). The script should:
1.  Read gig data from specified columns in the sheet (based on headers like those in `this-week.csv`).
2.  Check for potential duplicate gigs already existing in the LML system via a public API (`/gigs/query`).
3.  Format the data for non-duplicate gigs into the required "Clipper" style (multi-line `key: value` format ending with `---`).
4.  Authenticate using a session cookie (`_lml_session`) and submit the formatted data, including a CSRF token, to the LML admin upload endpoint (`/admin/uploads`).
5.  Upload gigs in small batches to improve error handling granularity.
6.  Provide clear feedback within the sheet regarding the status of each row (Uploaded, Suspected Duplicate, Failed) in dedicated columns, including highlighting failed batches.
7.  Integrate the trigger seamlessly into the existing custom menu structure within the sheet's Apps Script project.

## Core Logic Sources

*   **Data Formatting:** Implement the simplified "Clipper" style formatting based on the user-provided valid example and column names derived from `this-week.csv`. Handles multiple `set:`, `price:`, and `genre:` lines.
*   **Upload Mechanism:** Implement the authentication (session cookie), CSRF token fetching, and POST request structure based on the Python script `upload.py`.

## Key Features & Implementation Details

### 1. Duplicate Detection

*   **API:** Utilize the public LML API endpoint: `https://api.lml.live/gigs/query`.
*   **Frequency:** Call the API **once** per script execution (batch), not per row.
*   **Parameters:** Query using `location`, `date_from`, and `date_to`.
    *   `date_from`: Earliest date found in the sheet rows being processed.
    *   `date_to`: Latest date found + a configurable lookahead period (e.g., 30 days).
*   **Matching Criteria:** For each row in the sheet, compare against the fetched API results:
    *   Match `venue.id` (from API) with `venue_id` (from sheet).
    *   Match `date` (from API) with `date` (from sheet).
    *   Match `name` (from API) with `name` (from sheet) using a basic similarity check (case-insensitive, normalized comparison).
*   **Feedback:** If a potential duplicate is found for a sheet row, write "Suspected Duplicate: [Existing Gig Name] on [Date]" directly into the "Upload Status" column for that row. Rows marked as duplicates are skipped during the upload phase.

### 2. Data Formatting (Clipper Style)

*   **Function:** `formatRowForClipper` processes a single row.
*   **Input:** Sheet row data (both raw values for dates/times and display values for others), headers array.
*   **Output:** A multi-line string with `key: value` pairs based on column names (e.g., `venue_id`, `tickets`, `date`, `time`, `name`, `venue`, `status`, `information`, `internal_description`, `set1`-`set6`, `price1`-`price2`, `Genre Tag 1`-`Genre Tag 4`), ending in `---`. Handles missing optional values appropriately.
*   **Aggregation:** `formatDataAsClipper` iterates through rows *within a batch*, calls `formatRowForClipper`, and concatenates the resulting strings (separated by newlines) into a single payload string for the `lml_upload[content]` field for that batch.

### 3. Authentication & CSRF

*   **Authentication:** Use the `_lml_session` cookie value stored in Script Properties (`LML_SESSION_ID`). Include this in `Cookie` headers for authenticated requests (`/admin/uploads/new` and `/admin/uploads`).
*   **CSRF Token:**
    *   Fetch the token **once** before starting batch uploads by making a GET request to `/admin/uploads/new` (authenticated).
    *   Parse the HTML response using regex to extract the `authenticity_token` value from the input field.

### 4. Batch Upload Processing (Implemented - Option C)

*   **Batching:** Valid, non-duplicate rows are divided into small batches (size configured by `UPLOAD_CONFIG.BATCH_SIZE`, default 5).
*   **Iteration:** The script loops through each batch sequentially.
*   **Submission per Batch:**
    *   Make a POST request to `/admin/uploads` for *each batch*.
    *   Include required headers (`Cookie`, `Content-Type: application/x-www-form-urlencoded`, `Referer`, `Origin`, `User-Agent`).
    *   Include payload: `authenticity_token`, `lml_upload[source]` (generated label including user email), `lml_upload[content]` (the concatenated Clipper string *for the current batch*), `commit: 'Create Upload'`, and empty `lml_upload[venue_label]` / `lml_upload[venue_id]` fields.
    *   Set `followRedirects: false` and `muteHttpExceptions: true`.
*   **Response Handling per Batch:**
    *   Check response code (302 redirect usually indicates success).
    *   If successful (302), attempt to parse the Upload ID from the `Location` header.
    *   If failed (e.g., 422), attempt to parse error messages from the HTML response body.
*   **Continuation:** Processing continues to the next batch regardless of the success or failure of the current batch.

### 5. Feedback Mechanism (In-Sheet)

*   **Required Columns:** Ensure the following columns exist in the sheet (create if missing via `ensureFeedbackColumnsExist`):
    *   `Upload Status`
    *   `Upload ID`
    *   `Upload Error`
*   **Workflow:**
    1.  **Clear:** Before processing, clear existing content in the feedback columns for the relevant rows.
    2.  **Duplicates:** Write "Suspected Duplicate..." status during the duplicate check phase.
    3.  **Validation/Formatting Errors:** Write specific errors to `Upload Error` if basic validation or Clipper formatting fails for a row/batch.
    4.  **Upload Attempt (Per Batch):** For each batch:
        *   **Success:** Write "Uploaded" to `Upload Status` and the extracted ID (if any) to `Upload ID` for all rows in the batch. Clear `Upload Error` and background color.
        *   **Failure:** Write "Upload Failed" to `Upload Status` and the error details from the server/request to `Upload Error` for all rows in the batch. Set the background color of these rows to pink (`UPLOAD_CONFIG.ERROR_BACKGROUND_COLOR`).

### 6. Integration

*   **Script Files:** Logic resides primarily in `UploadLogic.gs`. Menu creation is handled in the user's existing active script file (e.g., `Code.gs`, `index.gs`). The local `samplegs.gs` file is informational only and not part of the active project.
*   **Menu Item:** The existing `onOpen` function in the active script file was modified to create a single "Live Music Locator" menu, incorporating the original items plus a new item: `Upload Gigs with Check` which calls the `uploadGigsWithCheck` function in `UploadLogic.gs`.


### 7. Configuration (Script Properties)

Store the following values in `File > Project properties > Script properties`:
*   `LML_BASE_URL`: Base URL of the LML application (e.g., `https://api.lml.live`).
*   `LML_SESSION_ID`: The value of the `_lml_session` cookie obtained from browser developer tools after logging into LML admin (requires periodic manual updates).
*   `SHEET_NAME`: Exact name of the sheet (tab) containing gig data (e.g., "Gig data entry - This week").
*   `DATA_RANGE`: Cell range covering the data rows/columns, excluding headers (e.g., "E2:Y2000"). Must include all columns needed for formatting and duplicate checks.
*   `DUPLICATE_LOOKAHEAD_DAYS`: Number of days into the future to check for duplicates via the API (e.g., `30`).
*   `QUERY_LOCATION`: Default location parameter for the `/gigs/query` API (e.g., "melbourne").

## High-Level Workflow

1.  User opens spreadsheet; `onOpen` runs, creating the "Live Music Locator" menu.
2.  User clicks "Upload Gigs with Check".
3.  `uploadGigsWithCheck` function starts in `UploadLogic.gs`.
4.  Validate Script Properties configuration.
5.  Ensure feedback columns exist in the target sheet.
6.  Clear previous feedback in columns for the data range.
7.  Read data (raw and display values) and headers from the configured sheet/range.
8.  Perform duplicate check:
    a. Determine date range.
    b. Call `/gigs/query` API once.
    c. Compare sheet rows to API results.
    d. Write "Suspected Duplicate..." status to sheet.
9.  Filter out rows marked as duplicates or having validation errors (e.g., invalid date).
10. If valid, non-duplicate rows exist:
    a. Fetch CSRF token **once** from `/admin/uploads/new`. Handle errors.
    b. **Loop through valid rows in small batches:**
        i. Format rows *in the current batch* into a Clipper string payload. Handle formatting errors for the batch.
        ii. Submit Clipper data payload for the *current batch* to `/admin/uploads` via POST. Handle submission errors for the batch.
        iii. Parse response (success/failure, potential Upload ID, error message) for the batch.
        iv. Write final status ("Uploaded"/"Upload Failed"), ID, error message, and background color back to the sheet for rows *in the current batch*.
11. Provide summary alert/log message to the user detailing batch processing results.

## Testing

*   User configures Script Properties accurately.
*   User runs the script via the menu item.
*   User observes sheet feedback columns and script execution logs (`Extensions > Apps Script > Executions`) for results and errors.
*   User updates `LML_SESSION_ID` property when the session expires.