# n8n Workflows: Email Summarizer + Slack Query Bot

This repository contains two exported n8n workflows used together to summarize emails and to query summarized email records from Slack.

Workflows included

- `Large_and_Small_email_summerizer (3).json`
  - Name in n8n: "Large and Small email summerizer"
  - Purpose: Ingest incoming Microsoft Outlook emails, classify them by size, summarize small/medium emails using OpenAI and chunk-and-summarize large email threads. Results are appended to a Google Sheet and posted into a Slack channel.
  - Trigger: `microsoftOutlookTrigger` (polls every minute in this export)
  - Key nodes:
    - Microsoft Outlook Trigger (`Microsoft Outlook Trigger2`) — watches mailbox
    - HTTP Request (`Get Full Email Body2`) — fetches full message via Graph API
    - Code nodes (`Classify Email Size3`, `Split Into Chunks3`, etc.) — clean/parse email, chunking, prompt building
    - HTTP Request nodes for OpenAI (`Summarize Each Chunk3`, `Create Final Summary3`, `OpenAI Small/Medium Email3`)
    - Google Sheets (`Append Large Email to Sheet2`, `Append to Google Sheet2`) — write summarized results to sheet
    - Slack (`Slack Large Email Notification3`, `Slack Notification2`) — post summaries to Slack
  - Credentials referenced (names from export):
    - Microsoft Outlook OAuth2: "Microsoft Outlook account 3"
    - OpenAI: "OpenAi account 2" (used for GPT model calls)
    - Google Sheets OAuth2: "Google Sheets account 2"
    - Slack OAuth2: "Slack account 7"
  - Important static values found in the export:
    - Google Sheet ID: `1h3pTq_tEb-vfAcROmJgS7Jqub2S0SEOM7TQKGx3sxYE` (sheet: Sheet1)
    - Slack channel ID used for notifications: `C09GZ3H9UVC` (cached name: `akd_coms_emails`)
    - OpenAI models used: `gpt-4o-mini` (for chunk and final summarization), and the small/medium summarizer also uses `gpt-4o-mini`.

- `My_workflow_7.json`
  - Name in n8n: "My workflow 7"
  - Purpose: Respond to Slack app mentions (bot) and query the Google Sheet of summarized emails using a LangChain agent + OpenAI model, then reply in Slack.
  - Trigger: `slackTrigger` (app_mention in channel `C09GJL07BHC`, cached name `akd_com_bot`)
  - Key nodes:
    - Slack Trigger (`Slack Trigger`) — listens for app mentions
    - LangChain Agent (`AI Agent`) — parses user query, constructs tool calls
    - LangChain/OpenAI model (`OpenAI Chat Model`) — model used by the agent (export shows `gpt-4.1-mini`)
    - Google Sheets Tool (`Get row(s) in sheet in Google Sheets`) — queries the same sheet used by the summarizer
    - Slack (`Send a message`) — posts reply into Slack
  - Credentials referenced (names from export):
    - Slack API: "Slack account 10"
    - OpenAI: "OpenAi account 2"
    - Google Sheets OAuth2: "Google Sheets account 2"

How to import these workflows into n8n

1. Open your n8n instance and go to Workflows -> Import.
2. Upload the JSON files (`Large_and_Small_email_summerizer (3).json` and `My_workflow_7.json`).
3. After import, open each workflow and:
   - Re-assign credentials to your n8n credential objects (the exported credential names are shown in each node, but those IDs are from the original system). Do NOT paste credential IDs; instead choose the proper credential entry in your n8n instance.
   - Check node positions and activate the workflow if desired.

Required credentials & recommended scopes

- Microsoft Outlook / Microsoft Graph OAuth2
  - Credential type: Microsoft Outlook OAuth2 (Microsoft Graph)
  - Minimum scopes: Mail.Read (reads message), Mail.ReadWrite (optional if modifying); adjust to the least privilege needed.

- OpenAI
  - Credential type: OpenAI API key
  - Used by: chunk summarization and short summaries. Monitor usage to control cost.

- Google Sheets OAuth2
  - Credential type: Google Sheets OAuth2
  - Minimum scopes: `https://www.googleapis.com/auth/spreadsheets` (and `drive.file` or appropriate drive scope if needed to access sheets)
  - The export appends rows to the spreadsheet ID listed above.

- Slack OAuth2 / Bot token
  - Credential type: Slack OAuth2 (bot)
  - Recommended bot scopes: `chat:write`, `channels:read` (or `conversations:read`), `channels:history`, `app_mentions:read`, `users:read`.
  - For posting messages into channels and receiving mentions, ensure the bot is installed in the target workspace and channel.

Security & cost notes

- Do not commit real API keys into this repo. n8n stores credentials encrypted in its database; when you import the workflow you'll need to configure credentials in the n8n UI.
- OpenAI usage can be expensive for large emails. The workflow chunks long emails and uses the `gpt-4o-mini` / `gpt-4.1-mini` models in the exported configuration — adjust model and temperature according to cost/performance needs.

Testing the workflows

- Large and Small email summarizer
  1. Make sure `Microsoft Outlook account` credential is set and has scope to read messages.
  2. Send a test email to the inbox watched by n8n (or set the Outlook trigger to run manually). Use both short emails and long threads to exercise both paths.
  3. Check that a new row is appended to the Google Sheet and that a Slack message is posted to the configured channel.
  4. Inspect node executions in n8n for errors; check the HTTP requests to OpenAI for responses.

- Slack query bot (My workflow 7)
  1. Ensure the bot credential is configured and the bot is a member of the channel `akd_com_bot`.
  2. In Slack, mention the bot with a test query, such as:
     - "@bot find emails from alice@example.com"
     - "@bot show recent project emails"
  3. Confirm the bot replies with results fetched from the Google Sheet.

Troubleshooting

- No messages appearing in Slack or sheet
  - Verify credentials for Outlook, Slack, and Google Sheets are correctly configured.
  - Confirm the bot is in the right Slack channel and has the right scopes.
  - Confirm the Google Sheet ID is accessible with the configured Google credentials.

- OpenAI errors or empty responses
  - Check the OpenAI credential and quota/limits in your OpenAI account.
  - Look at the returned response payload in the n8n node to see model errors or messages.
  - For very long emails: ensure the chunk size is appropriate; the workflow currently chunks in ~2500-word segments before sending to the model.

- Permission/403 errors on Graph API or Google Sheets
  - Revisit scopes and re-consent credentials. For Graph API, Mail.Read is required. For Sheets, spreadsheets scope is required.

Assumptions & notes I made while generating this README

- I used the exported credential names and node names from the JSON to list which credentials the workflow references; you will need to map these to credentials in your n8n instance.
- The OpenAI model names (`gpt-4o-mini`, `gpt-4.1-mini`) are taken from the workflow export; these may be changed to other available models depending on your account.
- Slack channel IDs and Google Sheet IDs are included in the exports; keep those values if you intend to use the same sheet and channels. If you prefer a new sheet/channel, update the relevant nodes after import.

Next steps / Suggestions

- Add monitoring/alerting: add error-handling nodes or notifications when summarization fails.
- Add rate-limit handling / backoff to OpenAI calls for robustness.
- Consider storing raw message IDs and a small text preview to avoid duplicating work on re-runs.

If you want, I can:
- Generate a shorter "Quick Start" README that only includes the essentials (credentials + one quick test).
- Create an exported, sanitized version of the workflows with credential references removed or replaced by placeholder names.

---
Generated on: October 30, 2025
(README assembled from the exported workflow JSON files in this repo.)