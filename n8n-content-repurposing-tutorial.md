# n8n Content Repurposing Pipeline — Zero to Working Build

This guide assumes you've never opened n8n. Follow it top to bottom, in order. Don't skip ahead — each step depends on the last one working.

---

## Part 0: Get n8n Running

You have two options. Pick one.

### Option A: n8n Cloud (easiest — recommended for beginners)
1. Go to n8n.io and click **"Start free trial"** or **"Get started"**.
2. Sign up with email or Google.
3. You'll land in a hosted n8n instance at a URL like `yourname.app.n8n.cloud`. This is your workspace — no installation needed.

### Option B: Self-hosted (free, but more setup)
1. You need Node.js installed on your computer first (nodejs.org, download the LTS version).
2. Open a terminal and run: `npx n8n`
3. It will print a URL like `http://localhost:5678` — open that in your browser.

**For this tutorial, I'll assume Option A (n8n Cloud)** since it's zero-config. Everything below works identically either way.

---

## Part 1: Create Your Workflow

1. Once logged in, you'll see a dashboard. Click the **"+ New Workflow"** button (usually top-right, orange).
2. You'll land on a blank canvas — this is where you build. It's empty except for maybe a placeholder.
3. Top-left, click the workflow title (says "My workflow") and rename it: **"Content Repurposing Pipeline"**.
4. This canvas auto-saves as you go, but hit **Ctrl+S / Cmd+S** after big changes to be safe.

---

## Part 2: Add the Trigger (Form Trigger)

This is the very first node — what starts the workflow.

1. On the blank canvas, you'll see a **"+"** button in the middle, or "Add first step". Click it.
2. A panel opens on the right with a search bar. Type: **"Form"**
3. Click **"n8n Form Trigger"** from the results.
4. A node settings panel opens. Here's what to fill in:
   - **Form Title**: `Repurpose Content`
   - **Form Description**: `Paste a blog URL or YouTube URL`
   - Under **"Form Fields"**, click **"Add Field"**:
     - Field 1: **Field Label** = `Content Type`, **Field Type** = `Dropdown List`, then add two options: `blog` and `youtube`
     - Field 2 (click Add Field again): **Field Label** = `URL`, **Field Type** = `Text`
5. Leave other settings as default.
6. Click the **X** or click outside the panel to close it — the node is now added to your canvas, labeled "On form submission".

**What this does**: creates a public web form. When someone (you, for testing) fills it out and hits submit, the workflow runs with that data.

**To test it right now**: click the **"Test workflow"** button (top right, usually has a play icon). n8n will show you a form URL — open it in a new tab, fill in a real blog URL, choose "blog", and submit. Come back to n8n — you should see the node turn green with a checkmark, meaning data came through.

---

## Part 3: Fetch the Content (HTTP Request node)

1. Click the small **"+"** icon that appears on the right edge of your Form Trigger node (hovering over the node reveals it), OR click the "+" on the canvas and it'll auto-connect to the last node.
2. Search for **"HTTP Request"** and click it.
3. Settings to configure:
   - **Method**: `GET`
   - **URL**: click in the field, type `{{ $json.URL }}` — this pulls the URL the user typed into the form. (n8n's form fields are usually named exactly as you typed the label, so double check the exact field name — you can see it in the previous node's output data by clicking on the Form Trigger node and looking at the "Output" panel on the right.)
4. Leave everything else default for now.
5. Click outside to close the panel.

**Test it**: click "Test workflow" again (or the "Test step" button on just this node — a play icon appears when you hover over the node itself). Check the output panel — you should see raw HTML text come back under a field like `data`.

---

## Part 4: Extract Clean Text from HTML

1. Add another node after HTTP Request. Search **"HTML"** and choose **"HTML Extract"**.
2. Settings:
   - **Source Data**: choose "JSON" if asked, then point it at the HTTP Request's output field (usually `data`).
   - Under **"Extraction Values"**, click **Add Value**:
     - **Key**: `articleText`
     - **CSS Selector**: this depends on the website. For a first test, try `body` (grabs everything — messy but works for testing). Later you'll refine this to something like `article` or `.post-content` depending on the actual site's HTML structure.
3. Close the panel.

**Test it**: run the node. You should see a field called `articleText` with a big blob of text in the output.

> **Note on YouTube path**: n8n doesn't have a native "get YouTube transcript" node. For now, focus on getting the blog path working end-to-end first — we'll add the YouTube branch afterward using an HTTP Request to a transcript API. Don't try to do both paths at once as a beginner; get one working, then copy the pattern.

---

## Part 5: Clean the Text (Code node)

1. Add a node after HTML Extract. Search **"Code"** and select it (choose **"Run Once for All Items"** mode if asked — it's usually the default).
2. You'll see a code editor. Delete whatever placeholder is there and paste this:

```javascript
let text = $input.first().json.articleText;

// Collapse whitespace
text = text.replace(/\s+/g, ' ').trim();

// Remove bracketed artifacts like [Music]
text = text.replace(/\[.*?\]/g, '');

// Cap length so we don't blow past AI token limits
const MAX_CHARS = 6000;
if (text.length > MAX_CHARS) {
  text = text.substring(0, MAX_CHARS) + '...';
}

return [{
  json: {
    cleanedText: text
  }
}];
```

3. Close the panel.

**Test it**: run the node, check output — you should see one clean field: `cleanedText`.

**If it errors**: the most common issue is the field name doesn't match. Click on the HTML Extract node, look at its actual output field name in the right panel, and make sure `$input.first().json.articleText` in the code matches that exact name (case-sensitive).

---

## Part 6: Set Up Your AI Node

You need an API key before this step works.

### Getting an API key (OpenAI example)
1. Go to platform.openai.com, sign up/log in.
2. Click **"API Keys"** in the left sidebar (or under your account settings).
3. Click **"Create new secret key"**, name it "n8n", copy the key (starts with `sk-...`). You won't see it again, so save it somewhere safe.
4. Note: you need billing set up on your OpenAI account (add a card) for API calls to work — the free ChatGPT subscription is separate from API credits.

### Adding the credential in n8n
1. Back in n8n, add a new node after the Code node. Search **"OpenAI"** and select it.
2. It'll ask you to select or create a **Credential**. Click **"Create New Credential"**.
3. Paste your API key into the **"API Key"** field. Click **Save**.
4. Now configure the node itself:
   - **Resource**: `Text` or `Chat` (depending on n8n version — pick "Message a model" / Chat Completion)
   - **Model**: `gpt-4o` or `gpt-4o-mini` (mini is cheaper, good enough for this)
   - **Response Format**: set to `JSON Object` if the option is available (important — forces clean JSON back)
   - **Messages**: add two messages:
     - **System message**:
       ```
       You are a social media repurposing assistant. Given source content, generate exactly three outputs. Respond ONLY with valid JSON, no markdown, no preamble, in this exact structure:
       {
         "linkedin_post": "string, 150-250 words, professional tone, 1-2 relevant hashtags",
         "twitter_thread": ["tweet 1", "tweet 2", "tweet 3"],
         "instagram_caption": "string, 100-150 words, casual tone, 5-8 hashtags at the end"
       }
       ```
     - **User message**: `{{ $json.cleanedText }}`
5. Close the panel.

**Test it**: run the node. Output should show a `message.content` (or similar) field containing a JSON-looking string.

---

## Part 7: Parse the AI's Response (Code node)

1. Add another **Code** node after the AI node.
2. Paste:

```javascript
const raw = $input.first().json.message?.content 
  || $input.first().json.content 
  || $input.first().json.text;

let parsed;
try {
  const cleaned = raw.replace(/```json\n?|\n?```/g, '').trim();
  parsed = JSON.parse(cleaned);
} catch (e) {
  throw new Error('AI response was not valid JSON: ' + raw.substring(0, 200));
}

return [{
  json: {
    linkedinPost: parsed.linkedin_post,
    twitterThread: parsed.twitter_thread,
    instagramCaption: parsed.instagram_caption
  }
}];
```

3. Close panel, run the node.

**If it errors with "raw.replace is not a function"**: the field path is wrong. Click the AI node, expand its output in the right panel, find exactly where the text response lives (it might be nested differently depending on n8n version), and adjust `$input.first().json.message?.content` to match.

---

## Part 8: Send to Slack for Review

### Setting up Slack access
1. You need a Slack workspace where you're allowed to add apps (use a test workspace if unsure).
2. In n8n, add a node after the parsing Code node. Search **"Slack"**.
3. Click **"Create New Credential"** — n8n will walk you through connecting via OAuth (a popup asks you to log into Slack and authorize). Approve it.
4. Configure the node:
   - **Resource**: `Message`
   - **Operation**: `Send`
   - **Channel**: pick from the dropdown (e.g., `#general` or a channel you made like `#content-review`)
   - **Message Text**: click the field and enter (or better, add a Code/Set node right before this one to build a formatted string — see below)

### Formatting the Slack message nicely
Add one more **Code** node before the Slack node:

```javascript
const item = $input.first().json;

const twitterFormatted = item.twitterThread
  .map((tweet, i) => (i + 1) + '. ' + tweet)
  .join('\n\n');

const slackText = '*New content ready for review* 📝\n\n' +
  '*LinkedIn:*\n' + item.linkedinPost + '\n\n' +
  '*Twitter Thread:*\n' + twitterFormatted + '\n\n' +
  '*Instagram Caption:*\n' + item.instagramCaption;

return [{ json: { slackText } }];
```

Then in the Slack node's **Message Text** field, click it and type `{{ $json.slackText }}`.

**Test it**: run the whole workflow end to end (click "Test workflow" from the very start). Check your Slack channel — you should get a formatted message with all three pieces of content.

---

## Quick Checklist Before You Start Clicking

- [ ] n8n account created (cloud or local)
- [ ] OpenAI API key created + billing enabled
- [ ] Slack workspace ready, willing to connect an app
- [ ] A test blog URL ready to paste into the form

## Order to Build (don't jump around)
1. Form Trigger → test
2. HTTP Request → test
3. HTML Extract → test
4. Code (clean) → test
5. AI node → test
6. Code (parse) → test
7. Code (format) + Slack → test
---

If any single step throws an error, stop there, copy the exact error message, and bring it back to me — I'll tell you exactly which field or setting to fix rather than you guessing.
