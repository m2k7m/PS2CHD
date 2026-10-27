# PS2 ISO to CHD to VikingFile

A manually triggered GitHub Actions workflow that downloads a direct ISO link, converts it with `chdman`, verifies the CHD, and uploads it to your VikingFile account. All processing code is embedded in `.github/workflows/ISOTOCHD.yaml`; no repository checkout or additional Python packages are required.

## Configure secrets

Open **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret | Value |
| --- | --- |
| `ISO_URL` | A direct HTTPS download URL for the ISO. Signed URLs and access tokens in the URL are supported. |
| `VIKINGFILE_USER_HASH` | Your VikingFile account's user hash: the `user` parameter described in the API documentation. This is not your password or a file hash. |

The workflow does not print the source URL, account hash, API error bodies, or temporary upload URLs. People who can modify the workflow can write code that reads its secrets, so restrict repository access appropriately. The final CHD download link appears in the run summary and is visible to people who can read the run. Use a private repository if those results should remain private.

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

The embed includes the output filename, current stage, elapsed time, and a link to the Actions run. On completion it turns green and includes the VikingFile download link, CHD size, total elapsed time, and individual durations for downloading, compression, verification, and upload. Times use `HH:MM:SS`; stage durations measure the operations and exclude the immediate stage-change notification request. Total elapsed time includes notification overhead and starts when processing begins, after tool installation. These are elapsed-time updates, not percentage estimates. Editing a Discord message updates its contents; it does not generate a new push notification for every stage.

The original ISO URL remains hidden in all messages because it is a secret. The starting embed says the task is for the configured ISO link and identifies the output filename. It never reveals the source URL or webhook URL. A handled processing failure turns the embed red and preserves the timings recorded so far. Force cancellation, runner shutdown, or failure before processing starts may leave the last status unchanged.

Telegram still sends a single completion message with the filename, size, download link, and stage durations. Both Telegram secrets are required. Use your numeric user ID and open the bot's private chat to send `/start` before running the workflow. If both services are configured, both receive the final result. The link and timings always remain in the Actions run summary.

With neither service configured, the result is printed in the Actions log. Invalid notification settings or failed final delivery produce a warning and also print the result without failing a successful upload. Telegram and Discord work independently. Credentials and raw notification errors are not printed. An uncertain initial webhook POST is not retried to avoid duplicate messages; failed edits can recover on the next periodic update, stage change, or final update. No replacement message is created automatically if the original is deleted. This feature sends links and status messages, not the CHD file itself.

## Run the workflow

1. Open **Actions → ISO to CHD to VikingFile → Run workflow**.
2. Set `filename`, for example `MyGame`. The `.chd` extension is added automatically. Use 1–100 ASCII letters, digits, dots, underscores, or hyphens, starting with a letter or digit. Do not include sensitive information.
3. Select `dvd` for an original DVD, or `cd` for a CD ISO with 2048-byte sectors. Disc type is not inferred from file size.
4. Start the workflow and wait for downloading, conversion, verification, and upload to finish.
5. Open the run's **Summary** and follow **VikingFile download**.

To process another game, update the `ISO_URL` secret and start a new run. The filename and disc type are ordinary visible inputs. Pushes and pull requests do not trigger processing; the workflow runs manually only.

## Upload behavior and limits

- Converts with `chdman createdvd` or `chdman createcd`, then runs `chdman verify` before uploading.
- Uses VikingFile's documented multipart API: `get-upload-url`, PUT requests for individual parts, and `complete-upload`. The `user` parameter associates the completed upload with your account. Parts stream from disk, with up to three attempts per part.
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
