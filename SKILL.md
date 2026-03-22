---
name: hashnode-blog-writer
version: 1.1.0
description: >
  Write high-quality English blog posts and publish to Hashnode with cover images.
  Two workflows: (A) Write from scratch - Writer (Opus) drafts, Reviewer (Sonnet) checks, iterate until pass, then publish.
  (B) Sync existing - Extract from Jekyll post, upload cover, publish.
  Input is a Google Doc URL, local file, topic description, or existing Jekyll post slug.
  Use when user says "write a blog", "publish to hashnode", "sync to hashnode", "blog about X",
  or provides a Google Doc URL for blog conversion.
category: workflow-automation
author: skill-engineer
owner_agent: any agent with shell access, web search, and subagent capabilities
requires:
  skills:
    - openclaw-skill-glm-image-generator  # cover image generation
  tools:
    - web_search
    - exec
    - sessions_spawn
    - message
  env:
    - HASHNODE_API_KEY
    - HASHNODE_PUBLICATION_ID
    - HASHNODE_HOST
    - ZAI_API_KEY  # for cover image generation
  credentials:
    HASHNODE_API_KEY: "Required via environment only"
    HASHNODE_PUBLICATION_ID: "Required via environment only"
    HASHNODE_HOST: "Required via environment only"
---

# Hashnode Blog Writer Skill

## Overview

Dual-agent iterative workflow for writing and publishing high-quality English blog posts.

Runtime placeholders used below:

- `$WORKSPACE_DIR`: writable workspace for draft and review artifacts
- `$IMAGE_SKILL_DIR`: path to the installed image-generation skill directory

```
Source Material → Writer (Opus) → Draft → Reviewer (Sonnet) → Pass/Fail
                     ↑                                            |
                     └──── Revision Notes (if fail) ──────────────┘
                                                                  |
                                                           Pass → Cover Image → Publish
```

**Max iterations:** 2 (if still failing after 2 rounds, present to user with remaining issues)

---

## Voice & Style Rules (MANDATORY)

These are the operator's explicit preferences. Every blog post MUST follow them:

1. **First person throughout.** "I did X", "my agent", "I discovered". NEVER "a user", "they", "one might".
2. **Genuine, not translated.** Must read like native English, not translated from another language.
3. **Technical specifics included.** Always mention: OpenClaw version, model names (e.g., Claude Opus 4.6), exact tools used. This makes articles timely and credible.
4. **No personal info leaks.** NEVER mention: real name, employer, location, workplace, family members, financial details, or anything from USER.md.
5. **Timely framing.** "Just days ago", "last week" — not vague "earlier this year" or "recently".
6. **Opinionated.** Have a clear thesis. State it plainly. Don't hedge.

---

## Step 0: Determine Input Type

### A. NEW article (write from scratch)
→ Follow Steps 1-6 (Writer → Reviewer → Publish)

### B. EXISTING article (sync from Jekyll/GitHub)
→ Skip to Step 0.5 (Sync Workflow)

---

## Step 0.5: Sync Existing Article (Alternative Entry Point)

When source is a complete Jekyll post already published to GitHub Pages:

**Typical trigger:** "sync this article to hashnode", "publish to hashnode" (with no Google Doc URL)

**Location:** `/Users/liao/public-workspace/chunhualiao.github.io/_posts/YYYY-MM-DD-slug.md`

**0.5.1 Identify the post**
```bash
# User provides slug or title
SLUG="the-wisdom-stack-why-ai-agents-are-finally-making-timeless-principles-actually-w"

# Find the file
POST_FILE=$(ls /Users/liao/public-workspace/chunhualiao.github.io/_posts/*-${SLUG}.md)

if [ ! -f "$POST_FILE" ]; then
  echo "ERROR: Post not found for slug: $SLUG"
  exit 1
fi
```

**0.5.2 Extract metadata from frontmatter**
```bash
TITLE=$(grep "^title:" "$POST_FILE" | sed 's/^title: "//' | sed 's/"$//')
SLUG=$(grep "^slug:" "$POST_FILE" | sed 's/^slug: //')
COVER_PATH=$(grep "^cover_image:" "$POST_FILE" | sed 's/^cover_image: "//' | sed 's/"$//')
TAGS=$(grep "^tags:" "$POST_FILE" | sed 's/^tags: \[//' | sed 's/\]$//' | tr ',' '\n' | sed 's/^ *//' | sed 's/ *$//')

echo "Title: $TITLE"
echo "Slug: $SLUG"
echo "Cover: $COVER_PATH"
```

**0.5.3 Extract content (skip frontmatter)**
```bash
# Skip YAML frontmatter, get markdown content
awk '/^---$/{if(++c==2){f=1;next}}f' "$POST_FILE" > /tmp/blog-content.md

# Verify content exists
if [ ! -s /tmp/blog-content.md ]; then
  echo "ERROR: No content extracted"
  exit 1
fi

echo "Content extracted: $(wc -w /tmp/blog-content.md | awk '{print $1}') words"
```

**0.5.4 Handle cover image**
```bash
# Convert Jekyll path to filesystem path
if [[ "$COVER_PATH" == /assets/* ]]; then
  # Relative to repo root
  FULL_COVER_PATH="/Users/liao/public-workspace/chunhualiao.github.io${COVER_PATH}"
elif [[ "$COVER_PATH" == assets/* ]]; then
  FULL_COVER_PATH="/Users/liao/public-workspace/chunhualiao.github.io/${COVER_PATH}"
else
  # Already absolute or external URL
  FULL_COVER_PATH="$COVER_PATH"
fi

if [ -f "$FULL_COVER_PATH" ]; then
  cp "$FULL_COVER_PATH" /tmp/blog-cover.png
  echo "✅ Cover image ready: $FULL_COVER_PATH"
else
  echo "⚠️  WARNING: Cover image not found at $FULL_COVER_PATH"
  echo "Will attempt to generate one..."
  # Fall back to Step 5 image generation
fi
```

**0.5.5 Proceed to publish**
→ Skip to **Step 5** (Upload cover) then **Step 6** (Publish)

---

## Step 1: Acquire Source Material

Determine input type and fetch content:

### From Google Doc:
```bash
gog docs cat <DOC_ID> > /tmp/blog-source.md
```

### From local file:
```bash
cp <path> /tmp/blog-source.md
```

### From topic (no source):
Use web_search to gather material, save notes to `/tmp/blog-source.md`.

**Output:** `/tmp/blog-source.md` with full source content.

---

## Step 2: Writer Agent (Opus)

Spawn a Writer subagent with Opus model:

```
sessions_spawn(
  task: "<see Writer Prompt Template below>",
  mode: "run",
  model: "anthropic/claude-opus-4-6",
  runTimeoutSeconds: 300
)
```

### Writer Prompt Template

```
Write a high-quality English blog post based on the source material at /tmp/blog-source.md.

## MANDATORY Voice Rules
- First person throughout ("I", "my", "I discovered")
- NEVER use third person ("a user", "they", "one might")
- Include technical specifics: OpenClaw version (2026.2.26), model names
- NO personal identifying information (no name, employer, location)
- Timely: "just days ago", "last week" — never vague timeframes
- Opinionated: clear thesis, stated plainly

## Structure
- Punchy title (under 80 chars)
- Subtitle (one sentence, hooks the reader)
- Target 1200-1800 words by default
- Use 1800-2400 only for genuinely deep tutorials, references, or broad essays
- Exceed 2400 only when the material truly earns it
- Clear sections with ## headers
- Concrete examples, not abstract theory
- End with actionable takeaways

## Output
Save the blog post (markdown, NO YAML frontmatter) to:
  $WORKSPACE_DIR/blog-draft.md

The file must contain ONLY the blog content. No frontmatter, no metadata blocks.
```

---

## Step 3: Reviewer Agent (Sonnet)

Spawn a Reviewer subagent with Sonnet model:

```
sessions_spawn(
  task: "<see Reviewer Prompt Template below>",
  mode: "run",
  model: "anthropic/claude-sonnet-4-6",
  runTimeoutSeconds: 120
)
```

### Reviewer Prompt Template

```
You are a blog post reviewer. Read $WORKSPACE_DIR/blog-draft.md
and evaluate against EVERY criterion below. Be strict.

## Quality Gate Checklist

### Voice (HARD FAIL if any violated)
- [ ] First person throughout — no "a user", "they", "one might"
- [ ] Reads as native English — no translation artifacts (翻译腔)
- [ ] Opinionated — clear thesis stated in first 3 paragraphs

### Technical Specifics (HARD FAIL if missing)
- [ ] OpenClaw version mentioned (e.g., "OpenClaw 2026.2.26")
- [ ] Model names mentioned (e.g., "Claude Opus 4.6", "Claude Sonnet 4.6")
- [ ] Specific tools/features named where relevant

### Privacy (HARD FAIL if any violated)
- [ ] No real names or aliases from the operator's personal denylist
- [ ] No employer or organization names from the operator's personal denylist
- [ ] No location details from the operator's personal denylist
- [ ] No family references
- [ ] No email addresses or usernames

### Structure
- [ ] Title under 80 characters
- [ ] Subtitle present (one sentence)
- [ ] Word count fits the piece: 1200-1800 by default, 1800-2400 for deep tutorials/references, >2400 only with clear justification
- [ ] Has clear ## section headers
- [ ] Ends with actionable takeaway or summary
- [ ] No YAML frontmatter or metadata blocks in content

### Content Quality
- [ ] Opening hook in first paragraph
- [ ] Concrete examples (not just abstract theory)
- [ ] No filler phrases ("Great question!", "Let's dive in!", "In this article we will...")
- [ ] No unresolved TODOs or placeholders

## Output Format

Write your review to $WORKSPACE_DIR/blog-review.md:

```
## Review Result: PASS / FAIL

### Hard Fails (if any)
- [criterion]: [specific issue + line/paragraph reference]

### Soft Issues (suggestions, not blockers)
- [issue]: [suggestion]

### Revision Notes (if FAIL)
Specific, actionable instructions for the writer to fix each hard fail.
Do NOT rewrite the article — just describe what needs to change.
```
```

---

## Step 4: Iterate (if FAIL)

If reviewer returns FAIL:

1. Read `$WORKSPACE_DIR/blog-review.md`
2. Spawn Writer again with additional context:
   ```
   Revise $WORKSPACE_DIR/blog-draft.md based on reviewer feedback at
   $WORKSPACE_DIR/blog-review.md

   Fix ONLY the issues flagged. Do not rewrite sections that passed.
   Save the revised version to the same path: $WORKSPACE_DIR/blog-draft.md
   ```
3. Spawn Reviewer again
4. **Max 2 iterations.** If still failing, present to user with remaining issues.

---

## Step 5: Handle Cover Image

After reviewer PASS, handle the cover image. Three sources in priority order:

### Option A: Google Doc has cover image link (preferred)
If source is a Google Doc with a cover image link in the first lines:
```bash
# Extract Google Drive image ID from doc
DRIVE_ID=$(grep -o 'drive.google.com.*id=[^)]*' /tmp/blog-source.md | head -1 | sed 's/.*id=//')

if [ -n "$DRIVE_ID" ]; then
  # Download from Google Drive
  curl -L "https://drive.google.com/uc?id=$DRIVE_ID" -o /tmp/blog-cover.png
fi
```

### Option B: Generate cover image via Z.AI
```bash
SKILL_DIR=$IMAGE_SKILL_DIR
ZAI_API_KEY="<key>" python3 "$SKILL_DIR/scripts/generate.py" \
  "<descriptive prompt — NO Chinese text, scrapbook/craft style, visual metaphor for the topic>" \
  --provider zai --size 1920x1088 --output /tmp/blog-cover.png
```

**Cover image rules:**
- Landscape 1920x1088
- NO Chinese text (English-only or no text)
- Visual metaphor related to the article topic
- Scrapbook/craft paper style preferred

### Upload to Hashnode CDN

```python
import os, requests, json

# Step 1: Get presigned upload URL
query = '''mutation { createImageUploadURL(input: { contentType: "image/png" }) {
  presignedPost { url fields } } }'''
r = requests.post('https://gql.hashnode.com/',
    headers={'Content-Type': 'application/json', 'Authorization': os.environ['HASHNODE_API_KEY']},
    json={'query': query})
presigned = r.json()['data']['createImageUploadURL']['presignedPost']

# Step 2: Upload to S3
with open('/tmp/blog-cover.png', 'rb') as f:
    requests.post(presigned['url'], data=presigned['fields'],
                  files={'file': ('cover.png', f, 'image/png')})

# Step 3: Construct CDN URL from the key field
key = presigned['fields']['key']
cdn_url = f"https://cdn.hashnode.com/{key}"
```

**NEVER use Google Drive URLs in the final post.** X/Twitter crawlers cannot follow Drive redirects,
resulting in missing Open Graph preview cards. Always re-upload to a public CDN.

---

## Step 6: Publish to Hashnode

### Validate API key first

**CRITICAL:** Test API key before attempting to publish.

```bash
API_KEY="$HASHNODE_API_KEY"

# Validate key works
TEST=$(curl -s -X POST https://gql.hashnode.com/ \
  -H "Content-Type: application/json" \
  -H "Authorization: $API_KEY" \
  -d '{"query": "query { me { id username } }"}')

if echo "$TEST" | jq -e '.errors[] | select(.extensions.code == "UNAUTHENTICATED")' > /dev/null 2>&1; then
  echo "❌ ERROR: Hashnode API key expired or invalid"
  echo ""
  echo "Current key (first 8 chars): ${API_KEY:0:8}..."
  echo ""
  echo "To fix:"
  echo "1. Get new key from https://hashnode.com/settings/developer"
  echo "2. Update ~/.openclaw/openclaw.json:"
  echo "   \"HASHNODE_API_KEY\": \"<new-key>\""
  echo ""
  exit 1
fi

USERNAME=$(echo "$TEST" | jq -r '.data.me.username')
echo "✅ API key valid (user: $USERNAME)"
```

### Check for existing post

**CRITICAL:** Always check if a post with the target slug already exists.
If it exists, UPDATE it. NEVER create duplicates.

```bash
SLUG="<slug-from-title>"
HOST="$HASHNODE_HOST"

# Query for existing post
EXISTING=$(curl -s -X POST https://gql.hashnode.com/ \
  -H "Content-Type: application/json" \
  -H "Authorization: $API_KEY" \
  -d "{\"query\": \"query { publication(host: \\\"$HOST\\\") { post(slug: \\\"$SLUG\\\") { id } } }\"}")

POST_ID=$(echo "$EXISTING" | jq -r '.data.publication.post.id')

if [ "$POST_ID" != "null" ] && [ -n "$POST_ID" ]; then
  echo "Post exists (ID: $POST_ID). Will update instead of creating new."
  ACTION="update"
else
  echo "Post does not exist. Will create new."
  ACTION="create"
fi
```

### Generate slug from title
```python
import re
slug = re.sub(r'[^a-z0-9]+', '-', title.lower()).strip('-')[:80]
```

**CRITICAL:** Slug derives from TITLE, never from filename.

### Publish via API (create OR update)

```python
import json, os, requests

with open(os.path.join(os.environ['WORKSPACE_DIR'], 'blog-draft.md')) as f:
    content = f.read()

query = '''mutation CreateDraft($input: CreateDraftInput!) {
  createDraft(input: $input) { draft { id } }
}'''

payload = {
    "query": query,
    "variables": {
        "input": {
            "publicationId": os.environ['HASHNODE_PUBLICATION_ID'],
            "title": "<TITLE>",
            "subtitle": "<SUBTITLE>",
            "contentMarkdown": content,  # NO YAML frontmatter
            "slug": "<SLUG>",
            "coverImageOptions": {
                "coverImageURL": "<HASHNODE_CDN_URL>"
            },
            "tags": [
                {"name": "AI", "slug": "ai"},
                {"name": "Multi-Agent Systems", "slug": "multi-agent-systems"}
            ]
        }
    }
}

r = requests.post('https://gql.hashnode.com/',
    headers={'Content-Type': 'application/json', 'Authorization': os.environ['HASHNODE_API_KEY']},
    json=payload)
draft_id = r.json()['data']['createDraft']['draft']['id']

# Then publish the draft
pub_query = '''mutation PublishDraft($input: PublishDraftInput!) {
  publishDraft(input: $input) { post { id url slug } }
}'''
pub_payload = {
    "query": pub_query,
    "variables": {"input": {"draftId": draft_id}}
}
r2 = requests.post('https://gql.hashnode.com/',
    headers={'Content-Type': 'application/json', 'Authorization': os.environ['HASHNODE_API_KEY']},
    json=pub_payload)
post = r2.json()['data']['publishDraft']['post']
print(f"Published: {post['url']}")
```

### Post-publish verification

**MANDATORY:** Verify the post is actually published and accessible.

```bash
POST_URL="<url-from-publish-response>"
SLUG="<slug>"

# Wait for propagation
sleep 3

# Verify via API (more reliable than HTTP, Cloudflare may block)
VERIFY=$(curl -s -X POST https://gql.hashnode.com/ \
  -H "Content-Type: application/json" \
  -H "Authorization: $HASHNODE_API_KEY" \
  -d "{\"query\": \"query { publication(host: \\\"$HASHNODE_HOST\\\") { post(slug: \\\"$SLUG\\\") { id title publishedAt url coverImage { url } } } }\"}")

VERIFIED_URL=$(echo "$VERIFY" | jq -r '.data.publication.post.url')
PUBLISHED_AT=$(echo "$VERIFY" | jq -r '.data.publication.post.publishedAt')

if [ "$VERIFIED_URL" != "null" ] && [ "$VERIFIED_URL" = "$POST_URL" ]; then
  echo ""
  echo "✅ Published successfully!"
  echo ""
  echo "Title: $TITLE"
  echo "URL: $VERIFIED_URL"
  echo "Published: $PUBLISHED_AT"
  echo "Cover: $(echo "$VERIFY" | jq -r '.data.publication.post.coverImage.url')"
  echo ""
else
  echo "❌ Verification failed"
  echo "Expected URL: $POST_URL"
  echo "API returned: $VERIFIED_URL"
  exit 1
fi
```

---

## Step 7: Post to X (optional, if user requests)

Draft a tweet (under 280 chars with URL counting as 23):

```python
# Effective char count = len(text) - len(url) + 23
# Must be <= 280
```

Post via tweepy using credentials from `openclaw.json` → `env.X_CONSUMER_KEY` etc.

---

## Step 8: Translate for WeChat (optional)

If user requests Chinese version, hand off to `blog-to-wechat` skill:
```
Use blog-to-wechat skill with URL: <PUBLISHED_HASHNODE_URL>
```

---

## Anti-Patterns (things that went wrong before)

| Mistake | Prevention |
|---------|-----------|
| YAML frontmatter rendered as text on Hashnode | Content file has NO frontmatter. Metadata passed via API fields only. |
| Slug from filename ("blog-filtering-trap") | Slug always derived from title via regex. |
| Third-person voice ("a user asked") | Reviewer hard-fails any third-person reference. |
| "Earlier this year" vague timing | Reviewer checks for specific timeframes. |
| Missing cover image | Step 5 is mandatory before Step 6. |
| Missing OpenClaw/model versions | Reviewer hard-fails if not mentioned. |
| Personal info leaked | Reviewer checks for name, employer, location patterns. |
| Broken old URL after slug change | Never change slug after publishing + X posting. Get it right first time. |
| Main session blocked during generation | ALL generation done via subagents. Main stays responsive. |
| Cover image not showing in X card | ALWAYS upload to Hashnode CDN, NEVER use Google Drive URLs. Drive requires auth/redirects that X crawlers cannot follow. |
| **Duplicate posts created** | **ALWAYS query for existing post by slug FIRST. Update if exists, create only if new.** |
| **Using expired API key** | **Step 6: Validate API key BEFORE publishing. Test with `query { me { id } }` first.** |
| **Not verifying publish succeeded** | **Step 6: Query API after publish to confirm post exists and URL matches.** |
| **Can't sync existing Jekyll posts** | **Step 0.5: Added workflow to extract from Jekyll frontmatter + content.** |
| **Cover image path mismatch** | **Step 0.5: Handle both relative (`/assets/`) and absolute paths from Jekyll frontmatter.** |

---

## File Locations

| File | Purpose |
|------|---------|
| `/tmp/blog-source.md` | Source material (input) |
| `workspace/blog-draft.md` | Current draft (writer output) |
| `workspace/blog-review.md` | Current review (reviewer output) |
| `/tmp/blog-cover-final.png` | Cover image |
| `workspace/blog-<slug>.md` | Final published version (archived) |

---

## Publish Script

If you already have a local publish helper script, validate that it handles slug generation
and frontmatter correctly before using it. Otherwise prefer the inline Python approach in
Step 6.
