# Rate Test

Single-threaded HTTP rate-limit testing helper that combines client-IP headers with alternate address formats, replays a supplied raw request, and records every generated request and response.

## Overview

`rate_limit_bypass.sh` is intended for authorised testing of HTTP rate limiting and proxy/client-IP handling.

### What it does

- Reads the source HTTP request from `request.txt`.
- Reads candidate header names from `headers.txt`.
- Reads candidate IP/address representations from `formats.txt`.
- Combines every header with every format.
- Sends requests sequentially using `curl`.
- Preserves the request method, endpoint, headers, and body.
- Displays status, rate-limit state, timing, and redirects.
- Stores every generated request and response separately.
- Writes a summary to `results.csv`.

## Files

### Required files

Place these files in the same directory:

```text
rate_test.sh
request.txt
headers.txt
formats.txt
```

### Generated files and folders

The script creates:

```text
requests/
responses/
results.csv
```

## Usage

### Make the script executable

```bash
chmod +x rate_test.sh
```

### Run the script

```bash
./rate_test.sh METHOD https://host/path
```

Example:

```bash
./rate_test.sh POST https://portal.example.test/account/recovery/request
```

The HTTP method must be uppercase.

## request.txt

### Purpose

`request.txt` contains the source HTTP request that will be replayed.

Example:

```http
POST /account/recovery/request HTTP/2
Host: portal.example.test
Content-Type: application/json

{"username":"sample-user"}
```

### Validation

The command-line method, host, and endpoint must match the values in `request.txt`.

The script does not rewrite the source request.

### Host and Content-Length

`Host` and `Content-Length` are handled by `curl` when the request is sent.

## headers.txt

### Format

Add one candidate header name per line.

Include the colon, but do not add a trailing space.

Example:

```text
X-Forwarded-For:
X-Real-IP:
Forwarded:
True-Client-IP:
CF-Connecting-IP:
```

### Injection

The script adds the space between the header name and value automatically.

For example:

```text
X-Forwarded-For:
```

combined with:

```text
127.0.0.1
```

becomes:

```http
X-Forwarded-For: 127.0.0.1
```

## formats.txt

### Format

Add one candidate value or address representation per line.

Example:

```text
127.0.0.1
127.1
2130706433
::1
::ffff:127.0.0.1
for=127.0.0.1
for="127.0.0.1"
```

### Combination logic

Every non-empty header in `headers.txt` is combined with every non-empty value in `formats.txt`.

For example:

```text
40 headers × 125 formats = 5000 requests
```

## Request count warning

### Pre-run count

Before sending any requests, the script displays:

```text
Headers       : 40
Formats       : 125
Requests      : 5000
```

### Confirmation threshold

If the total is greater than 100 requests, the script displays a warning and asks for confirmation:

```text
WARNING: This test will send 5000 requests.
Do you want to continue? [y/N]
```

## Terminal output

### Columns

The console output uses:

```text
REQ | METHOD | HEADER | VALUE | STATUS | REMAINING | RESET | TIME | REDIRECT
```

### Colours

- `204` responses are green.
- `429` responses are red.
- `000` curl/network failures are yellow.
- Redirect values are light blue.
- Other status codes use the normal terminal colour.

## Rate-limit headers

### Remaining requests

The script checks for:

```text
X-RateLimit-Remaining:
```

and suffixed variants such as:

```text
X-RateLimit-Remaining-User:
X-RateLimit-Remaining-Account:
X-RateLimit-Remaining-Email:
```

### Reset timer

The script checks in this order:

```text
X-RateLimit-Reset:
X-RateLimit-Reset-<suffix>:
Retry-After:
```

Example:

```http
Retry-After: 60
```

is displayed as:

```text
RESET
60
```

## Redirects

### Source

The `REDIRECT` column is populated from the HTTP response `Location` header.

Examples:

```http
Location: /signin
```

or:

```http
Location: https://portal.example.test/signin
```

### Display

If a redirect exists, the redirect value is displayed in light blue.

If no `Location` header is present:

```text
-
```

is displayed.

## Request captures

### Folder

Every generated request is saved under:

```text
requests/
```

Examples:

```text
requests/request_00001.txt
requests/request_00002.txt
```

### Contents

Each request capture contains the source request with the current generated header/value pair injected before the blank line separating headers from the body.

## Response captures

### Folder

Every response is saved under:

```text
responses/
```

Examples:

```text
responses/response_00001.txt
responses/response_00002.txt
```

### Contents

Each response file contains:

```text
response headers

response body
```

## results.csv

### Purpose

`results.csv` provides a summary of the complete run.

### Columns

```text
request
method
target
header
format
status
remaining
reset
time
redirect
error
request_file
response_file
```

### Debugging

Use the `request_file` and `response_file` columns to inspect the exact request and full response for any interesting result.

## HTTP behaviour

### Request handling

Requests are sent sequentially, one at a time.

### HTTP version

The script uses HTTP/2.

### Timeouts

```text
Connect timeout : 5 seconds
Maximum time    : 15 seconds
```

### Redirect handling

Redirects are not automatically followed.

This keeps the original response visible and allows the `Location` value to be recorded in the `REDIRECT` column.

## Status 000

### Meaning

`000` is not an HTTP status code returned by the server.

It means `curl` failed to receive a valid HTTP response.

### Debugging

The associated curl error is displayed in the terminal and written to `results.csv`.

## Notes

### Input handling

- Blank lines in `headers.txt` are ignored.
- Blank lines in `formats.txt` are ignored.
- `request.txt` is not modified.

### Evidence

- Generated requests are retained.
- Full responses are retained.
- Summary results are retained in CSV format.

### Intended use

Use only against systems you are authorised to test.
