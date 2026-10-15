# AI Lead Qualification & Follow-up System (n8n)

An n8n workflow that takes a website enquiry, scores it, replies with a personalised email in seconds, alerts the team on Slack when a lead is hot, follows up automatically if a warm lead goes quiet, and logs every lead to Google Sheets.

## How it works

```
Lead form (webhook) -> Normalize Lead -> Gemini (summary + reply draft) -> Parse Score (rule-based tier)
  -> Google Sheets log -> Route by Tier
       hot  -> Slack alert + personalised email
       warm -> personalised email -> wait 2 days -> follow-up email
       cold -> nurture placeholder (connect Mailchimp)
```

The AI writes the summary and the reply email. **The hot / warm / cold decision is made by plain rules** in the `Parse Score` node, so the same lead always lands on the same branch and the rules are easy to adjust.

## Scoring rules

| Signal | Points |
|---|---|
| Message of 20+ characters | 20 |
| Asks for a quote, price, visit or availability | 20 |
| Budget contains an amount (any digit) | 30 |
| Timeline is soon (this month, within weeks, asap) | 20 |
| Company provided | 5 |
| Name provided | 5 |

- **Hot:** budget amount stated AND timeline is soon
- **Warm:** score of 30 or more
- **Cold:** everything else

## Files

- `ai-lead-qualification-workflow.json` - main workflow
- `error-alert-workflow.json` - sends a Slack message and an email when the main workflow fails
- `sample-leads.json` - one hot, one warm and one cold test payload

## Setup

1. In n8n, import `ai-lead-qualification-workflow.json` (menu > Import from file).
2. Create these credentials and attach them to the nodes:
   - **Gemini:** Header Auth, name `x-goog-api-key`, value is your key from Google AI Studio (free tier available). Check the model name in the `Gemini Qualify` URL against the models available to you.
   - **Google Sheets, Gmail, Slack:** connect with OAuth.
3. Create a Google Sheet with these headers in row 1:
   `name, email, company, budget, timeline, message, source, received_at, intent_summary, budget_fit, reply_subject, reply_body, score, tier, status`
   Then choose it in the `Log to CRM Sheet` node.
4. Create a `#leads` Slack channel and add your Slack app to it.
5. Import `error-alert-workflow.json`, connect Slack and Gmail, replace `YOUR_EMAIL@example.com`, then in the main workflow open menu > Settings > Error workflow and select it.
6. Publish the main workflow and copy the **Production URL** from the `Lead Webhook` node.

## Testing

Click **Execute workflow**, then POST one of the payloads in `sample-leads.json` to the **Test URL**:

```bash
curl -X POST "<your-test-webhook-url>" -H "Content-Type: application/json" -d '{"name":"Ade","email":"you@example.com","company":"Ade Interiors","budget":"₦2,000,000","timeline":"this month","message":"Need a full home renovation quote for a 4-bedroom flat in Lekki."}'
```

While testing, point the Gmail nodes at your own address and shorten the Wait node. Restore both before going live.

## Notes

- Credentials are not included in the exported files. Re-create them after importing.
- The Gemini free tier may use submitted data to improve Google's products. For real customer data, use a paid tier or another provider.
- Error workflows only fire on live (published) runs, not on manual test runs.
