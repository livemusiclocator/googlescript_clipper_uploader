# LML Gig Uploader for Google Sheets

This project provides a Google Apps Script (`UploadLogic.gs`) designed to upload gig information from a Google Sheet to the LML (Live Music Locator) platform via its API.

## Features

*   Reads gig data from a specified sheet and range.
*   Checks for potential duplicate gigs already existing in LML within a configurable date range.
*   Formats gig data into the required 'Clipper' format.
*   Uploads new (non-duplicate) gigs to LML in small batches (default size 5).
*   Provides feedback directly in the Google Sheet:
    *   Upload Status (Uploaded, Upload Failed, Suspected Duplicate).
    *   Upload ID (for successful batches).
    *   Upload Error messages (for formatting issues, API errors, etc.).
    *   Highlights rows belonging to failed upload batches with a pink background.
*   Includes the uploader's email address in the upload source label for tracking.
*   Skips processing completely empty rows.

## Setup

1.  **Copy Script:** Copy the code from `UploadLogic.gs` into the Google Apps Script editor associated with your Google Sheet (Tools > Script editor).
2.  **Set Script Properties:** Go to File > Project properties > Script properties in the Apps Script editor and add the following properties:
    *   `LML_BASE_URL`: The base URL of the LML API (e.g., `https://api.lml.live`).
    *   `LML_SESSION_ID`: The value of your `_lml_session` cookie after logging into the LML website/admin interface.
    *   `SHEET_NAME`: The exact name of the sheet tab containing your gig data (e.g., `Gig data entry - This week`).
    *   `DATA_RANGE`: The range containing the gig data, *excluding* the header row (e.g., `A2:G` if your data starts in row 2 and uses columns A to G).
    *   `DUPLICATE_LOOKAHEAD_DAYS`: How many days *after* the latest date in your sheet to check for duplicates in LML (e.g., `30`).
    *   `QUERY_LOCATION`: The location identifier used for the LML duplicate check query (e.g., `melbourne`).
3.  **Save Project:** Give your Apps Script project a name and save it.

## Usage

1.  **Prepare Data:** Ensure your Google Sheet has the necessary columns (e.g., `venue_id`, `date`, `time`, `name`, `tickets`, `internal_description`, `Venue`, `status`, `information`, `set1`, etc.) matching the headers expected by the script (see `formatRowForClipper` function). The header row should be directly above the `DATA_RANGE`.
2.  **Run Script:**
    *   Reload your Google Sheet. A new menu item (usually "LML Tools") should appear.
    *   Click the menu item and select "Upload Gigs with Check".
    *   Authorize the script if prompted.
3.  **Review Feedback:** Check the "Upload Status", "Upload ID", and "Upload Error" columns (the script adds these if they don't exist) and look for any pink-highlighted rows indicating failed batches.

## Files

*   `UploadLogic.gs`: The main Google Apps Script code.
*   `upload_script_plan.md`: Development notes and planning details.
*   `samplegs.gs`: (Potentially sample or older script code).
*   `this-week.csv`: (Likely sample input data).
*   `upload.py`: (Potentially related Python script, purpose unclear from context).