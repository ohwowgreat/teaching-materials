# Notion ↔ GitHub Sync: Build Instructions

When a lesson plan is merged to `main` on GitHub, this system automatically
updates the corresponding page in Notion. One-way: GitHub is the source of
truth, Notion reflects it.

---

## Overview of the flow

```
merge to main
  → GitHub Action triggers
  → detects which .md files changed
  → looks up matching Notion page ID from mapping file
  → calls Notion API to overwrite page content
```

---

## Step 1 — Create a Notion integration

1. Go to https://www.notion.so/my-integrations
2. Click **New integration**
3. Name it `github-sync`, set workspace to your teaching workspace
4. Under **Capabilities**, enable: Read content, Update content, Insert content
5. Click **Save** and copy the **Internal Integration Token** — you'll need this later

---

## Step 2 — Connect the integration to each Notion page

For every lesson plan page in Notion:

1. Open the page
2. Click the `...` menu (top-right)
3. Click **Connect to** → select `github-sync`

This must be done for every page you want synced. If you skip a page, the
script will leave it untouched.

---

## Step 3 — Record Notion page IDs

Each Notion page has a unique ID in its URL:

```
https://www.notion.so/My-Page-Title-<PAGE_ID>
```

The page ID is the last part — a 32-character string, e.g.:
`a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4`

Create a file at `docs/notion-page-map.json` in this repo mapping each `.md`
file path to its Notion page ID:

```json
{
  "art-appreciation/semester-1/03-food-and-ethics/01-eat-drink-man-woman.md": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "art-appreciation/semester-1/03-food-and-ethics/02-lunch-introduction.md": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
  "art-appreciation/semester-1/04-images-mediation-and-modernity/01-introduction.md": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6"
}
```

Add an entry for every lesson plan file. Any file not listed here will be
ignored by the sync script.

---

## Step 4 — Store the Notion token as a GitHub secret

1. In the `teaching-materials` repo, go to **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `NOTION_TOKEN`
4. Value: the token you copied in Step 1
5. Click **Add secret**

---

## Step 5 — Create the sync script

Create the file `scripts/sync_to_notion.py`:

```python
import os
import json
import sys
import re
import httpx

NOTION_TOKEN = os.environ["NOTION_TOKEN"]
HEADERS = {
    "Authorization": f"Bearer {NOTION_TOKEN}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json",
}

def md_to_notion_blocks(md_text):
    """Convert markdown text into a list of Notion block objects."""
    blocks = []
    for line in md_text.splitlines():
        if line.startswith("# "):
            blocks.append({"object": "block", "type": "heading_1",
                "heading_1": {"rich_text": [{"type": "text", "text": {"content": line[2:]}}]}})
        elif line.startswith("## "):
            blocks.append({"object": "block", "type": "heading_2",
                "heading_2": {"rich_text": [{"type": "text", "text": {"content": line[3:]}}]}})
        elif line.startswith("### "):
            blocks.append({"object": "block", "type": "heading_3",
                "heading_3": {"rich_text": [{"type": "text", "text": {"content": line[4:]}}]}})
        elif line.startswith("- "):
            blocks.append({"object": "block", "type": "bulleted_list_item",
                "bulleted_list_item": {"rich_text": [{"type": "text", "text": {"content": line[2:]}}]}})
        elif line.strip() == "":
            pass  # skip blank lines
        else:
            blocks.append({"object": "block", "type": "paragraph",
                "paragraph": {"rich_text": [{"type": "text", "text": {"content": line}}]}})
    return blocks

def clear_page(page_id):
    """Delete all existing blocks on a Notion page."""
    url = f"https://api.notion.com/v1/blocks/{page_id}/children"
    while True:
        r = httpx.get(url, headers=HEADERS)
        r.raise_for_status()
        data = r.json()
        for block in data["results"]:
            httpx.delete(f"https://api.notion.com/v1/blocks/{block['id']}", headers=HEADERS)
        if not data.get("has_more"):
            break

def append_blocks(page_id, blocks):
    """Append new blocks to a Notion page in batches of 100 (API limit)."""
    url = f"https://api.notion.com/v1/blocks/{page_id}/children"
    for i in range(0, len(blocks), 100):
        batch = blocks[i:i+100]
        r = httpx.patch(url, headers=HEADERS, json={"children": batch})
        r.raise_for_status()

def sync_file(file_path, page_id):
    with open(file_path, "r") as f:
        md_text = f.read()
    blocks = md_to_notion_blocks(md_text)
    print(f"  Clearing page {page_id}...")
    clear_page(page_id)
    print(f"  Writing {len(blocks)} blocks...")
    append_blocks(page_id, blocks)
    print(f"  Done: {file_path}")

if __name__ == "__main__":
    changed_files = sys.argv[1:]  # passed in from the GitHub Action
    with open("docs/notion-page-map.json") as f:
        page_map = json.load(f)

    for file_path in changed_files:
        if file_path in page_map:
            print(f"Syncing {file_path}...")
            sync_file(file_path, page_map[file_path])
        else:
            print(f"No Notion mapping for {file_path}, skipping.")
```

---

## Step 6 — Create the GitHub Action

Create the file `.github/workflows/sync-to-notion.yml`:

```yaml
name: Sync lesson plans to Notion

on:
  push:
    branches:
      - main
    paths:
      - 'art-appreciation/**/*.md'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 2  # needed to compare with previous commit

      - name: Get changed markdown files
        id: changed
        run: |
          FILES=$(git diff --name-only HEAD~1 HEAD -- 'art-appreciation/**/*.md' | tr '\n' ' ')
          echo "files=$FILES" >> $GITHUB_OUTPUT

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install httpx

      - name: Run sync script
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
        run: python scripts/sync_to_notion.py ${{ steps.changed.outputs.files }}
```

---

## Step 7 — Test it

1. Commit and push everything (`docs/notion-page-map.json`, `scripts/sync_to_notion.py`, `.github/workflows/sync-to-notion.yml`) to `main`
2. Make a small edit to any lesson plan `.md` file and merge it to `main`
3. Go to the **Actions** tab in the repo and watch the workflow run
4. Check the corresponding Notion page — it should reflect the change within seconds

---

## Limitations to be aware of

- **Markdown conversion is basic.** The `md_to_notion_blocks` function handles
  headings, bullet points, and paragraphs. Bold, italic, links, tables, and
  code blocks are not converted — they will appear as plain text. This can be
  improved later with a library like `mistletoe` if needed.
- **One-way only.** Edits made directly in Notion will be overwritten next time
  the GitHub file is merged. GitHub is the source of truth.
- **Page must be connected.** If Step 2 is skipped for a page, the API call
  will fail with a 403 error. The script will print an error but continue
  syncing other files.
