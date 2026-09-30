# n8n Sales Ops Automations

Two n8n workflows that take repetitive work off a B2B sales team and keep the CRM (HubSpot) up to date automatically. An LLM handles the unstructured part (reading free text, judging fit, drafting emails); plain workflow logic handles routing and CRM writes.

Both are built for the same context as my [AI Knowledge Assistant for manufacturing SMEs](https://github.com/Alex-lab-1/rag-sme-assistant): Italian manufacturing SMEs, where requests arrive as messy free text and sales time is scarce.

| | WF1 – RFQ Intake | WF2 – Lead Research & Outreach |
|---|---|---|
| **Problem** | Quote requests arrive incomplete; sales wastes time chasing missing details before even starting a quote | Reps spend time researching companies that were never a fit, and write generic cold emails |
| **Trigger** | Web form (the "request a quote" page) | Manual run on a list of target companies |
| **AI step** | Extracts quantity, material, drawing format, deadline; detects what is missing | Reads the company website, scores fit 1–10 with a reason, writes a personalized email |
| **Output in HubSpot** | Complete request → contact + deal. Incomplete → contact + task with a reply asking only for the missing data | Fit ≥ 6 → company + follow-up task with the draft email. Fit < 6 → discarded |

---

## WF1 – RFQ Intake

![WF1 flow](Screenshot%20github/01_wf1_flow.png.png)

**Flow:** Form → OpenAI (structured extraction, JSON mode) → data preparation → HubSpot contact upsert → *complete?* → deal **or** task.

**How it works**
- The customer describes the part in their own words. No rigid form with 15 fields: the model extracts the structured data from free text.
- A request counts as complete when it has the four things a supplier needs to quote: quantity, material, drawing format, delivery date.
- Complete requests become a deal in the pipeline, with a summary and a confirmation reply ready to send.
- Incomplete requests become a high-priority task with a draft reply that asks **only** for the missing information.
- Contacts are upserted by email, so a returning customer does not create a duplicate.

![AI extraction](Screenshot%20github/02_wf1_ai_extraction.png.png)

| Deal created | Task with draft reply |
|---|---|
| ![Deal](Screenshot%20github/03_wf1_hubspot_deal.png.png) | ![Task](Screenshot%20github/04_wf1_hubspot_task.png.png) |

## WF2 – Lead Research & Outreach

![WF2 flow](Screenshot%20github/05_wf2_flow.png.png)

**Flow:** Lead list → fetch website → clean HTML to text → OpenAI (fit score, reasoning, pitch angle, email) → *fit ≥ 6?* → HubSpot company + task **or** discard.

**How it works**
- The model uses **only** the website text: it is instructed not to invent facts about the company and not to claim customers or results.
- Every lead gets a score **and** a written reason, including rejected ones. A CRM needs to know why a lead was dropped, not just that it was.
- Emails are formal, in first person, and must open with a concrete detail from the prospect's website (a product, process or certification). Generic phrases are explicitly banned.
- Leads scoring 8+ get high priority, 6–7 medium.

---

## Design decisions

- **Human in the loop.** Neither workflow sends emails. They prepare drafts inside HubSpot tasks; a person reviews and sends. For B2B outreach and customer replies, a wrong automated email costs more than the minutes saved.
- **Qualify before spending time.** The fit filter in WF2 exists so reps never open a task for a company that was never going to buy.
- **Least-privilege access.** HubSpot is connected through a Service Key limited to the CRM scopes actually used (contacts, companies, deals). No credentials are stored in this repository.
- **Fail safe, not silent.** If a website cannot be read or the model's answer cannot be parsed, the lead gets score 0 and a stated reason instead of breaking the run.
- **Tested on both paths.** Each workflow was run on cases that should pass and cases that should fail (incomplete request; off-target company such as a beauty salon) to check the routing, not just the happy path.

## Stack

n8n (self-hosted) · OpenAI API (`gpt-4o-mini`, JSON mode) · HubSpot CRM API v3 · JavaScript (n8n Code nodes)

## Run it yourself

1. Install n8n: `npm install -g n8n`, then `n8n start` and open `http://localhost:5678`.
2. Import the files in [`workflows/`](workflows/) (menu ⋯ → Import from File).
3. Create two credentials: **OpenAI API** and **HubSpot App Token** (a HubSpot Service Key with contacts, companies and deals read/write scopes).
4. WF1: click *Execute workflow* and fill in the form. WF2: replace the placeholders in the *Lista Lead* node with real companies and run.

## What I would add in production

- Run on a server (Docker on a VPS, or n8n Cloud) instead of a local instance, so the form is always online.
- WF1: trigger from a shared inbox as well as the form; attach the uploaded drawing to the deal.
- WF2: read the lead list from a Google Sheet or a HubSpot list; deduplicate companies by domain before creating them.
- An error-notification workflow (Slack or email) and a simple log of runs, scores and costs to monitor quality over time.

---

Built by **Alex Bof** · [LinkedIn](https://linkedin.com/in/alexbof) · HubSpot Sales Hub Software Certified
