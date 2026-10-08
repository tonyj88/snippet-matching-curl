# Check a file for snippet matches with curl

This guide shows how to send one source file to a Black Duck SCA server's snippet-matching API using only `curl`. No Python or other tooling is needed.

The API returns the open source files that match parts of your code. For each match it reports the project, release, license, matched file, and matching line ranges. Snippet matches are evidence to review. They are not a complete inventory or a legal conclusion. No matches does not prove that no code was reused.

The sample file, `samples/DeserializationTask.java`, is from [OWASP WebGoat](https://github.com/WebGoat/WebGoat) (GPL-2.0). It is included so you can test the steps against a file that is known to produce matches.

## What you need

- `curl` (included with Windows 10 and later, and with most Linux distributions)
- Network access to your Black Duck SCA server
- An API token. See [Create an API token](#create-an-api-token).
- A UTF-8 text source file with **300–50,000 non-whitespace characters**. The server rejects other sizes, and curl does not check the size for you.

Send source text only to a Black Duck server that your organization approves for this purpose. Check your source-sharing rules first.

## Create an API token

Log in to Black Duck SCA, open the user menu, and select **My Access Tokens**. Select **Create New Token** and choose the scope agreed with your administrator. Copy the token while it is visible and store it securely. See the [Black Duck SCA token instructions](https://docs.blackduck.com/r/blackduck/2026.7/black-duck-documentation/managing-user-access-tokens.html).

The snippet-matching entitlement, role, and token scope the API requires are not documented here. Check with your administrator if the request returns an authorization error.

## Linux

Run these steps in order in the same terminal window. Each step is one block to copy and paste. The commands work in bash and zsh.

**Step 1.** Set your server URL and the file to check. Replace the server URL first. Then type the token at the `API token:` prompt and press Enter. The token is not shown while you type.

```sh
BLACKDUCK_URL='https://your-blackduck-server'
FILE='samples/DeserializationTask.java'
printf 'API token: '; stty -echo; read -r BLACKDUCK_API_TOKEN; stty echo; echo
```

**Step 2.** Exchange the API token for a bearer token. The last line prints the bearer token's length. If it prints `0`, open `auth.json` to see the error.

```sh
curl --silent --show-error --request POST \
  "$BLACKDUCK_URL/api/tokens/authenticate" \
  --header "Authorization: token $BLACKDUCK_API_TOKEN" \
  --header 'Accept: application/vnd.blackducksoftware.user-4+json' \
  --output auth.json --write-out 'HTTP %{http_code}\n'
BEARER=$(sed -n 's/.*"bearerToken" *: *"\([^"]*\)".*/\1/p' auth.json)
echo "Bearer token length: ${#BEARER}"
```

**Step 3.** Send the file. `HTTP 200` means success, and `matches.json` holds the matches. For any other status, `matches.json` holds the error body.

```sh
curl --silent --show-error --request POST \
  "$BLACKDUCK_URL/api/snippet-matching" \
  --header "Authorization: Bearer $BEARER" \
  --header 'Content-Type: text/plain' \
  --header 'Accept: application/vnd.blackducksoftware.bill-of-materials-6+json' \
  --data-binary "@$FILE" \
  --output matches.json --write-out 'HTTP %{http_code}\n'
```

To check another file, set `FILE` to its path and run step 3 again. Copy `matches.json` first, because step 3 overwrites it.

**Step 4.** Delete the credentials when you finish.

```sh
rm -f auth.json
unset BEARER BLACKDUCK_API_TOKEN
```

## Windows PowerShell

The commands use `curl.exe`, because `curl` in Windows PowerShell is an alias for a different command. Run these steps in order in the same PowerShell window.

**Step 1.** Set your server URL and the file to check. Replace the server URL first.

```powershell
$env:BLACKDUCK_URL = 'https://your-blackduck-server'
$file = 'samples\DeserializationTask.java'
$secret = Read-Host 'API token' -AsSecureString
$env:BLACKDUCK_API_TOKEN = [System.Net.NetworkCredential]::new('', $secret).Password
```

The third line does not ask for your token yet. It shows the prompt text `API token` and waits. **Do not replace `'API token'` with your token.** Leave the command as written, press Enter, and then paste your token at the prompt. The token is not shown while you type or paste, and nothing appears after you press Enter. The fourth line then reads the token into `$env:BLACKDUCK_API_TOKEN`.

**Step 2.** Exchange the API token for a bearer token. If the status is not 200, the error body is printed.

```powershell
$status = curl.exe --silent --show-error --request POST `
  "$env:BLACKDUCK_URL/api/tokens/authenticate" `
  --header "Authorization: token $env:BLACKDUCK_API_TOKEN" `
  --header 'Accept: application/vnd.blackducksoftware.user-4+json' `
  --output auth.json --write-out '%{http_code}'
Write-Host "HTTP $status"
if ($status -eq '200') { $bearer = (Get-Content auth.json -Raw | ConvertFrom-Json).bearerToken } else { Get-Content auth.json }
```

**Step 3.** Send the file. `HTTP 200` means success, and `matches.json` holds the matches. For any other status, the error body is printed.

```powershell
$status = curl.exe --silent --show-error --request POST `
  "$env:BLACKDUCK_URL/api/snippet-matching" `
  --header "Authorization: Bearer $bearer" `
  --header 'Content-Type: text/plain' `
  --header 'Accept: application/vnd.blackducksoftware.bill-of-materials-6+json' `
  --data-binary "@$file" `
  --output matches.json --write-out '%{http_code}'
Write-Host "HTTP $status"
if ($status -ne '200') { Get-Content matches.json }
```

To check another file, set `$file` to its path and run step 3 again. Copy `matches.json` first, because step 3 overwrites it.

**Step 4.** Delete the credentials when you finish.

```powershell
Remove-Item -LiteralPath auth.json
$bearer = $null
Remove-Item Env:BLACKDUCK_API_TOKEN
```

## Read the response

`matches.json` is raw JSON. Matches are grouped by license family under `snippetMatches`. Each match contains:

| Field | Meaning |
| --- | --- |
| `projectName`, `releaseVersion` | The open source project and release that matched |
| `licenseDefinition` | The license name, SPDX ID, and display name |
| `matchedFilePath` | The file in that project that matched |
| `regions` | `sourceStartLines` and `sourceEndLines` refer to your file. `matchedStartLines` and `matchedEndLines` refer to the matched file. |

With no matches, the response is still HTTP 200, and `snippetMatches` is an empty object (`{}`).

To view the JSON in a readable form on Linux, run `python3 -m json.tool matches.json`.

## Keep the credentials safe

`auth.json` contains a credential. Delete it after use, and do not share authentication responses. Other users on the same computer can see curl headers in process listings, so prefer a script that reads the token from an environment variable for routine or automated checks.

## Troubleshooting

- **Auth returns a non-200 status:** Check the server URL and token. Open `auth.json` to see the error body.
- **401 on step 3:** The bearer token may have expired. Run step 2 again, then step 3. If it persists, ask your administrator to check the token and its permissions.
- **Files outside the size range:** The file must have 300–50,000 non-whitespace characters. Curl does not check this, so count first. Files that are too small or too large are not matched reliably.
- **Non-UTF-8 file:** Convert the file to UTF-8 first.
- **TLS or proxy errors:** Check DNS, connectivity, and trust with your administrator. Do not turn off certificate verification for routine use.
