# PS2 ISO to CHD to VikingFile

A manually triggered GitHub Actions workflow that downloads a direct ISO link, converts it with `chdman`, verifies the CHD, and uploads it to your VikingFile account. All processing code is embedded in `.github/workflows/ISOTOCHD.yaml`; no repository checkout or additional Python packages are required.

## Configure secrets

Open **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret | Value |
| --- | --- |
| `ISO_URL` | A direct HTTPS download URL for the ISO. Signed URLs and access tokens in the URL are supported. |
| `VIKINGFILE_USER_HASH` | Your VikingFile account's user hash: the `user` parameter described in the API documentation. This is not your password or a file hash. |

The workflow does not print the source URL, account hash, API error bodies, or temporary upload URLs. People who can modify the workflow can write code that reads its secrets, so restrict repository access appropriately. When Telegram or Discord settings are present, the final result and download link are sent only to the configured services, without a GitHub run summary or a result printed in the log. With no notification settings, the result appears in the GitHub summary and log. Normal processing logs, filenames, workflow inputs, and generic warnings remain visible in GitHub.

## Optional Telegram and Discord notifications

Add the optional **Actions secrets** for either service, both services, or neither:

| Service | Required secrets |
| --- | --- |
| Telegram | `TELEGRAM_BOT_TOKEN` and `TELEGRAM_USER_ID` |
| Discord | `DISCORD_WEBHOOK_URL` |

Discord uses an incoming webhook URL. In Discord, open **Edit Channel → Integrations → Webhooks → New Webhook**, copy its URL, and save it as the `DISCORD_WEBHOOK_URL` Actions secret. Creating the webhook requires the Manage Webhooks permission. Use a normal text channel and the copied URL without query parameters.

The webhook creates one colored embed when processing starts, then edits that same message at each stage and approximately every 30 seconds during long operations:

```text
Task started → Downloading → Compressing → Verifying → Uploading → Completed
```

The embed includes the output filename, detected disc type and detection reason, current stage, elapsed time, and a link to the Actions run. Download and upload fields show the running **average speed in MiB/s** and transferred MiB, refreshed with the message. Upload traffic includes bytes sent again on a retry; it measures data passed to the network connection, not confirmed server-side progress. A MiB is 1,048,576 bytes. Final averages remain visible after completion, along with the VikingFile download link, CHD size, total elapsed time, and individual durations for downloading, compression, verification, and upload. Times use `HH:MM:SS`; stage durations measure the operations and exclude the immediate stage-change notification request. Total elapsed time includes notification overhead and starts when processing begins, after tool installation. These are elapsed-time and average-speed updates, not percentage estimates. Editing a Discord message updates its contents; it does not generate a new push notification for every stage.

The embed also shows the **original ISO size**, **compressed CHD size**, and **space saved in bytes and as a percentage**. ISO size appears after downloading; CHD size and savings appear after compression and remain visible during verification, upload, and the final result. Values use MiB/GiB with exact byte counts. Savings are calculated as `(ISO bytes - CHD bytes) / ISO bytes × 100`; if the CHD is larger, the embed reports the increase instead. The Output field includes the destination folder before the filename, such as `PS2/game.chd`; root uploads show only `game.chd`. There are no separate folder or Source fields in Discord. These details are included in Telegram's completion message, or in GitHub's summary when no notifications are configured.

The original ISO URL remains hidden in all messages because it is a secret. The starting embed identifies the output filename and destination folder. It never reveals the source URL or webhook URL. A handled processing failure turns the embed red and preserves the timings recorded so far. Force cancellation, runner shutdown, or failure before processing starts may leave the last status unchanged.

Telegram sends a single completion message with the filename, destination folder, disc type, sizes, savings, transfer speeds, download link, and stage durations. Both Telegram secrets are required. Use your numeric user ID and open the bot's private chat to send `/start` before running the workflow. If both services are configured, both receive the final result. The GitHub run summary is left empty when any notification setting is present.

With neither service configured, the result is written to the GitHub summary and printed in the Actions log. Invalid, partial, or failed notification settings produce a generic warning without publishing the result or download link in GitHub; check your VikingFile account if delivery fails. This also applies when only one Telegram secret is set. A successful upload still succeeds if notifications fail. Telegram and Discord work independently. Credentials and raw notification errors are not printed. An uncertain initial webhook POST is not retried to avoid duplicate messages; failed edits can recover on the next periodic update, stage change, or final update. No replacement message is created automatically if the original is deleted. This feature sends links and status messages, not the CHD file itself.

## Run the workflow

1. Open **Actions → ISO to CHD to VikingFile → Run workflow**.
2. Set `filename`, for example `MyGame`, `MyGame.iso`, or `MyGame.chd`. All three produce `MyGame.chd`. Use 1–160 letters (including Unicode), digits, spaces, dots, parentheses, brackets, apostrophes, ampersands, plus signs, commas, underscores, or hyphens, starting with a letter or digit. Leading and trailing whitespace is trimmed. Paths, slashes, control characters, and other punctuation are rejected. Do not include sensitive information.
3. Leave `disc_type` on **auto** for automatic detection. The detected type and reason appear in the log and Discord embed. You can still select `dvd` or `cd` as a manual override when you know the original media type.
4. Set the optional GitHub input `viking_folder` to the destination folder in your VikingFile account. The input is prefilled with **PS2**. You can enter a nested path such as `PS2/Europe`; this is a text field, not a live list of account folders. Clear it, leave it unset, or enter `root` (case insensitive) or `...` to upload directly to your account root. The workflow omits VikingFile's `path` parameter for root uploads. Other paths must be relative and up to 255 characters, with no empty segments, backslashes, or `..` segments. Enter the folder here and only the game name in `filename`.
5. Start the workflow and wait for downloading, conversion, verification, and upload to finish.
6. Read the result in Discord or Telegram if configured. Otherwise, open the run's **Summary** and follow **VikingFile download**.

To process another game, update the `ISO_URL` secret and start a new run. The filename, disc type, and destination folder are ordinary visible inputs. The folder is passed as VikingFile's `path` parameter when completing the upload. Pushes and pull requests do not trigger processing; the workflow runs manually only.

## Upload behavior and limits

### Automatic disc detection

Detection examines the downloaded ISO before compression:

- A UDF `NSR02` or `NSR03` volume-recognition descriptor selects DVD.
- A `CD-XA001` marker in the ISO primary volume descriptor selects CD.
- An image larger than 1 GiB selects DVD because it exceeds the capacity of an original PS2 CD.
- A smaller ISO without either marker is ambiguous, so the workflow stops with an explanation and asks you to rerun with an explicit `dvd` or `cd` override. Conflicting markers also require an override.

An ISO does not always preserve enough information to identify its original physical medium. Small size alone never selects CD. This automatic detection handles common PS2 disc images, rather than guaranteeing identification of every custom or modified image. CD mode supports single-track 2048-byte ISO images; it cannot reconstruct tracks or audio omitted from an ISO dump. .

### Transfer behavior

- Converts with `chdman createdvd` or `chdman createcd`, then runs `chdman verify` before uploading.
- Uses VikingFile's documented multipart API: `get-upload-url`, PUT requests for individual parts, and `complete-upload`. The `user` parameter associates the completed upload with your account. Parts stream from disk, with up to three attempts per part. Signed HTTPS storage URLs returned by the official API may belong to external storage providers such as OVH; they are accepted without requiring a `vikingfile.com` hostname. Upload redirects remain disabled, and signed URLs are never logged. Exhausted upload retries report a safe HTTP status or connection error without exposing the URL.
- Completion checks accept file sizes returned as JSON integers or decimal strings; VikingFile's live API returns a string even though its example shows a number. The size and filename must still match the local CHD. A metadata mismatch explicitly states that completion was requested and asks you to check your account before rerunning, since the file may already exist.
- VikingFile's **Upload remote file** feature fetches an existing public URL. The newly converted CHD exists on the Actions runner, so this workflow uploads it directly. Sending the original ISO URL to that feature would upload the ISO without converting it.
- Accepts raw ISO9660 images only, not ZIP/RAR/7z archives, download pages, or BIN/CUE images. The source must work without interactive login or cookies and remain accessible during the download. Redirects must remain HTTPS.
- Limits ISO downloads to 9 GiB and checks free disk space for both the ISO and CHD. The job timeout is 350 minutes. Large or slow jobs can still fail because of runner storage, time limits, or GitHub/VikingFile account limits.
- Failed downloads restart from the beginning on the next run. Only individual upload parts are retried automatically. Upload completion is not automatically retried because the first request may have succeeded despite a delayed response. Check your VikingFile account before rerunning after a completion failure.
- Temporary files are removed on normal exit or handled failure. Hosted runner environments are temporary. ISO and CHD files are not saved as GitHub artifacts. Forced cancellation may interrupt cleanup, and incomplete uploads may leave parts on VikingFile.
- Use disc images you have the right to process and upload.

## References

- [VikingFile API](https://vikingfile.com/api)
- [GitHub: Actions secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
- [GitHub: Workflow syntax and location](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [MAME: chdman](https://docs.mamedev.org/tools/chdman.html)
- [Telegram: sendMessage](https://core.telegram.org/bots/api#sendmessage)
- [Discord: Execute and edit webhook messages](https://docs.discord.com/developers/resources/webhook)
- [PCSX2: Disc image formats](https://pcsx2.net/docs/setup/discs/)
