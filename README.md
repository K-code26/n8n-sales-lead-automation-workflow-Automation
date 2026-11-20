# AI Lead Qualification Automation(n8n + Gemini)
An automated workflow that reads every new business enquiry, uses AI to judge how serious the buyer is (hot, warm or cold), and takes the right action: a personalised reply and instant alert for hot leads, a follow-up for warm leads, and a nurture list for cold leads. Every lead is logged in the google sheet.

Note: This is a personal learning project built on simulated data. It has not been deployed in a real business.

## The problem

Slow replies lose sales. A business that gets a few hundred enquiries a week cannot answer them all instantly, so a serious buyer (for example, a E-commerce chain asking for 500 units next month) can wait in the same queue as someone who is "just looking". This project automates the sorting so the most promising leads get attention first.

## What it does

1. A lead submits an enquiry through a **Google Form**, which saves to a **Google Sheet**.
2. **n8n** detects the new row and sends the enquiry to **Google Gemini**.
3. Gemini scores the lead using a written rubric and returns a score, a reason, a short summary and a suggested next step.
4. The workflow routes the lead by score:
   - **Hot:** personalised email to the lead.
   - **Warm:** softer follow-up email.
   - **Cold:** added to a **Nurture** tab in the Google Sheet. No immediate email.
   - **Unexpected result:** logged as "needs manual review" so nothing is lost.
5. Every lead's score, reason, action taken and processing time is written back to the sheet.
   
## Workflow diagram

![Workflow screenshot](screenshots/workflow.png)

## Tech stack

| Tool | Purpose |
|---|---|
| n8n (Cloud) | Workflow automation |
| Google Gemini API | Lead scoring and summaries |
| Google Forms and Sheets | Lead capture and logging |
| Gmail | Automated replies |

## How the scoring works

The AI follows a written rubric:

| Score | Meaning |
|---|---|
| **Hot** | Clear need AND (a stated budget OR a purchase timeline within about 30 days) |
| **Warm** | Real interest, but budget or timeline is missing, or the timeline is 1 to 3 months |
| **Cold** | Vague, just browsing, student or research, spam, or unrelated |

The model is asked to return **only valid JSON**, which the workflow splits into separate fields (`score`, `reason`, `summary`, `suggested_next_step`) so the next steps can use them.

<details>
<summary>View the scoring prompt</summary>

```
You are a lead qualification assistant for [BUSINESS NAME], which sells
[WHAT YOU SELL] to [WHO YOUR CUSTOMERS ARE].

Score the enquiry below as hot, warm or cold using these rules:
- hot: clear need AND (a stated budget OR a purchase timeline within 30 days)
- warm: genuine interest but budget or timeline is missing, or timeline is 1-3 months
- cold: vague, just browsing, student or research, spam, or unrelated

Return ONLY valid JSON with no extra text, in exactly this shape:
{"score": "hot|warm|cold", "reason": "one sentence",
 "summary": "max 20 words for the sales team",
 "suggested_next_step": "one short action"}

Enquiry:
Name: {{ $json["Name"] }}
Message: {{ $json["[YOUR MESSAGE COLUMN]"] }}
```

</details>

## Testing and results

I tested the workflow on **10 simulated leads**, including edge cases such as spam, empty fields and non-English messages, and compared the AI's score with my own manual score.

**Result:** the AI's score matched my manual label on **9 of 10** leads.

| # | Lead (summary) | My score | AI score | Match |
|---|---|---|---|---|
| 1 | bulk order for salon chain, budget stated | Hot | Hot | Yes |
| 2 | Asking for wooden toys, no timeline] | Warm | cold | No |
| 3 | just looking | Cold | Cold | Yes 

"AI is rigid with the conditions given as qualifying category, if any of that does not match then it is not categorised as a Hot Lead"

## Reliability features

- **Retries:** the AI and Gmail steps retry automatically if a service is briefly unavailable.
- **Fallback path:** leads with unexpected scores are logged for manual review instead of disappearing.
- **Score cleaning:** scores are converted to lower case and trimmed before routing, so "Hot " is treated as "hot".
- **Audit log:** every lead's status, action taken and processing date is recorded in the sheet.

## Set it up yourself

**You will need:** an n8n account, a Google account, a Google Gemini API key.

1. **Create the form and sheet.** Make a Google Form with fields for name, email, phone, message and anything else you want the AI to use. Link it to a Google Sheet and name the tab **Leads**.
2. **Add log columns** to the right of the form columns: `Score`, `Reason`, `Action taken`, `Processed At`, `Processed`. Add a second tab called **Nurture**.
3. **Import the workflow.** In n8n, choose *Import from file* and select `workflow/lead-qualification.json`.
4. **Create your own credentials** for Google Sheets, Gmail, and Gemini. Credentials are not stored in the exported file.
5. **Update the placeholders:** the spreadsheet and tab, and the column names in the expressions so they match your sheet exactly.
6. **Activate the workflow** and submit a test lead through your form.

> **Security:** never commit API keys to GitHub. Use dummy data only when testing.

## Repository structure

```
├── README.md
├── workflow/
│   └── Sales Lead Automation.json
├── screenshots/
│   ├── workflow.png

```

## Limitations

- Tested on a small set of simulated leads, so the accuracy figure may not hold on real enquiries.
- The AI can misjudge borderline leads, so in real use a human should review important ones.
- The nurture list is only stored, not yet followed up automatically.
- Not connected to a CRM, and no real customer data or consent process was involved.

## Future improvements

- Send a scheduled nurture email to the people on the Nurture list.
- Receive leads from a real website form using a webhook.
- Enrich leads with company information before scoring.
- Add a weekly summary email for the sales manager.
- Track which lead source produces the most hot leads.

## What I learned

- Writing a clear scoring rubric matters more than complicated logic.
- AI output must be forced into a fixed format (JSON) before other steps can use it.
- Small details such as exact node names, field names and capital letters decide whether an automation works.
- Testing against my own judgment showed me where the AI was reliable and where it was not.

## Author
Komal Bani
