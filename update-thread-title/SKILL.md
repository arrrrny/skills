---
name: update-thread-title
description: Quickly fix or update the current Zed thread title when the auto-generated title is incorrect, mangled, or no longer reflects the conversation focus.
---

# Update Thread Title

Use this skill when the user says the thread (conversation) title in Zed is wrong, mangled, cut off, or outdated. This can happen when:

- Zed's auto-generated title was truncated or garbled
- The conversation has shifted topics and the title no longer fits
- A code edit or paste inadvertently changed the title metadata
- The title shows raw text, incomplete content, or encoding issues

## Instructions

1. **Read the conversation context** — scan the most recent messages and code changes to understand the actual topic.

2. **Propose a concise, accurate title** (2–8 words, plain language, no markdown) that reflects the current conversation's primary goal or topic. For example:
   - "Add clearCookies method to CookieManager"
   - "Fix thread title auto-update bug"
   - "Refactor authentication middleware"

3. **Suggest the title to the user** with a clear action they can take, since only the user can set the thread title in Zed's UI (e.g., right-click the tab → "Rename" or click the title in the chat panel). Example phrasing:

   > Suggested title: **Add clearCookies method to CookieManager**
   > 
   > To update, right-click the tab and select "Rename", or click the title in the chat panel and paste this in.

4. If the user confirms, you're done. If they want a refinement, iterate on the title until they're satisfied.
