# 🔁 n8n Content Repurposing Pipeline

Turn one blog post or YouTube video into a **LinkedIn post, Twitter/X thread, and Instagram caption** — automatically, with a single form submission.

Built in [n8n](https://n8n.io) using a Form Trigger → HTTP Request → HTML Extract → Code (clean) → OpenAI → Code (parse) → Slack pipeline. No coding experience required — this is a complete, beginner-friendly, step-by-step build guide.

---

## ✨ What it does

1. You submit a form with a content type (`blog` or `youtube`) and a URL.
2. The workflow fetches the raw content and strips it down to clean text.
3. An AI model (OpenAI GPT-4o / GPT-4o-mini) rewrites it into three ready-to-post formats.
4. Everything lands in a Slack channel, formatted and ready for review.

```
Form Trigger → HTTP Request → HTML Extract → Code (clean text)
      → OpenAI (generate 3 formats) → Code (parse JSON)
      → Code (format message) → Slack (post for review)
```

## 🧰 Prerequisites

- [ ] n8n account (Cloud or self-hosted — both covered below)
- [ ] OpenAI API key + billing enabled on your OpenAI account
- [ ] A Slack workspace where you can install apps
- [ ] A test blog URL to try it with

## 🚀 Getting Started

### 1. Run n8n
**Option A — n8n Cloud (recommended for beginners)**
Sign up at [n8n.io](https://n8n.io) → you'll get a hosted instance at `yourname.app.n8n.cloud`. Zero setup.

**Option B — Self-hosted (free)**
Requires [Node.js](https://nodejs.org). Run:
```bash
npx n8n
```
Open the printed URL (usually `http://localhost:5678`).

### 2. Build the workflow
Full instructions, exact field names, and troubleshooting notes for every node are in [`n8n-content-repurposing-tutorial.md`](./n8n-content-repurposing-tutorial.md). Build and test each node **in order** — each step depends on the last:

| Step | Node | Purpose |
|---|---|---|
| 1 | Form Trigger | Collects content type + URL |
| 2 | HTTP Request | Fetches the raw page/content |
| 3 | HTML Extract | Pulls clean article text |
| 4 | Code | Strips whitespace, artifacts, caps length |
| 5 | OpenAI | Generates LinkedIn/Twitter/Instagram copy as JSON |
| 6 | Code | Parses the AI's JSON response |
| 7 | Code + Slack | Formats and posts the final message |

> 📌 **Note:** This build focuses on the blog path first. A YouTube transcript branch can be added afterward using an HTTP call to a transcript API — see the tutorial for details.

### 3. Test end-to-end
Click **Test workflow**, submit a real blog URL through the generated form, and check your Slack channel for the formatted output.

## 🐛 Troubleshooting

- **Field name mismatches** are the most common error — n8n names form/node outputs based on your exact labels. Always check the "Output" panel on the previous node before referencing a field in a Code node.
- **AI response not valid JSON** — confirm `Response Format: JSON Object` is set on the OpenAI node.
- Stuck on something else? Open an issue with the exact error message and the node it happened on.

## 🗺️ Roadmap

- [ ] YouTube transcript branch
- [ ] Support for additional platforms (Threads, Facebook)
- [ ] Auto-scheduling instead of manual Slack review

## 📄 License

MIT — free to use, modify, and share.

## 🙌 Contributing

PRs and suggestions welcome, especially around the YouTube branch and additional platform formats.
