# Writing Style Guide

This document captures the distinctive voice and patterns evident in your published posts.

## Tone & Voice

**Personal and experiential.** Your writing draws from 20+ years of professional experience and frames things through a lens of "what I've learned works in practice."
- Use "I" and "my" — position yourself as the guide sharing direct experience
- Be conversational without being unprofessional. Contractions are welcome ("I've never needed", "doesn't adequately")
- Casual phrases add warmth: "hands down", "pretty flexible", "by no means exhaustive"
- Explain the *why* alongside the *what*. A feature isn't just good; it's good because it solves a specific real-world constraint

**Practical over theoretical.** Your audience is working developers who care about solutions that actually ship.
- Acknowledge trade-offs explicitly ("Image widget crop is useful if you need... but that in turn adds workload...")
- Emphasize developer experience: what makes a tool or module pleasant to use day-to-day
- Include disclaimers for work you've created ("Disclaimer: This is one of my modules")
- Reference constraints: client needs, editorial workload, environment-specific issues

## Structure

**Opening paragraph:** Hook with context or a problem statement. Keep it tight (1–3 sentences).
- Example: "At Numiko we are blessed with amazing clients..."
- Example: "There are a number of drupal 8 modules which I almost always start a project with..."

**Length varies by content type:**
- **Short posts** (1–2 paragraphs): Module recommendation, quick project highlight, or event recap. Lead with a summary, optionally add a link.
- **Medium posts** (3–5 sections): Portfolio pieces or detailed module rundowns. Use section headers (`##`) to guide the reader. Include external links to live work or resources.
- **Longer posts** (6+ sections): Deep dives into a feature or workflow. Use subsections (`###`) for granularity. Break up long paragraphs with lists.

**Lists and emphasis:**
- Use bullet points (`*` or `-`) for related options or features
- Use bold (`**module-name**`) for terminology or product names
- Use markdown headers (`##`, `###`) liberally; they improve scannability
- Code blocks or inline backticks for technical references (but keep them minimal in narrative posts)

## Content Patterns

### Technical Posts (Drupal modules, Wagtail, etc.)
1. One-sentence summary of what the module does
2. Brief explanation of the problem it solves or the use case
3. Why you use it (personal experience, or what makes it stand out)
4. Link to the project/module
5. Optional: Bonus recommendations or related tools

Structure: "This module is..." → "It solves..." → "Why I use it..." → link

### Portfolio / Client Work Posts
1. Personal opening (how you felt, context, or the significance)
2. Your role and what you built (without over-explaining the tech)
3. Client mission or principles (what they care about)
4. Key accomplishments or moments of pride
5. Links to live site and company write-up

Tone: Celebrate the team and the client's mission. Your technical decisions should serve their goals.

### Game Dev / Personal Projects
1. Concept in one sentence
2. How it works (gameplay, mechanics, tools used)
3. Bullet list of features or mechanics
4. Embedded video or link to demo

Tone: Playful and experimental. Acknowledge if it's a learning project or throwaway prototype.

## Language & Style Notes

**Spelling & accuracy:**
- Proofread for typos (your posts have occasional ones like "almsot" instead of "almost"; watch for these)
- Use British English spellings where appropriate ("colour", "summarise") or pick US and be consistent
- Check module/product names for correct capitalization and punctuation

**Links:**
- Link generously. Every module, framework, or external resource should be clickable.
- Prefer text links within paragraphs over standalone URLs
- Use descriptive link text that tells the reader what they're clicking on

**Disclaimers:**
- Always mention when recommending your own work or modules you've created
- Example: "*Disclaimer: This is one of my modules*"

**Contractions and colloquialisms:**
- Contractions are fine and expected ("I've", "doesn't", "that's")
- Casual phrases work well, but avoid slang that won't age well
- "So" as a sentence starter is common in your writing; it's fine, but use it judiciously

**Emphasis and variety:**
- Vary sentence length. Mix punchy observations with longer, more detailed explanations
- Use bold sparingly for emphasis or key terms
- Avoid excessive punctuation; your style is already conversational enough

## Post Metadata (Front Matter)

Every post includes:
- `title` — Clear, descriptive (2–6 words typical). "Accessible Media Embed" not "A Module"
- `description` — One sentence that would fit in a social media preview. Explain what the post is about.
- `date` — ISO 8601 format (YYYY-MM-DDTHH:MM:SS+TZ)
- `image` — Path to a featured image (e.g., `images/drupal-accessible-media.png`). Gives visual identity to posts in listings.
- `tags` — Comma-separated list relevant to content (e.g., `["drupal", "drupalmodule", "php"]`). Enables discovery.
- `type` — Always `"post"`
- `draft` — `false` when published; `true` while writing

Pick tags that reflect the post's content, technology, and category. Recurring tags: `drupal`, `drupalmodule`, `php`, `gamedev`, `unity`, `csharp`, `wagtail`, `numiko`, etc.

## What Not to Do

- **Don't be preachy.** Share what works for you; don't dictate how others should build.
- **Don't bury the lede.** Get to the point in the first paragraph.
- **Don't over-explain basic concepts.** Your audience is developers; assume technical familiarity.
- **Don't link spam.** External links should add value, not feel promotional.
- **Don't neglect the image.** Each post should have a meaningful featured image that works at small scale.
- **Don't write unsourced claims.** If you say "most developers prefer X", ground it in your experience ("In my experience...").

## Consistency Checklist

Before publishing:
- [ ] Opening paragraph is tight and hooks the reader
- [ ] Post has a clear purpose (recommendation, reflection, tutorial, showcase)
- [ ] External links are included where relevant
- [ ] Front matter is complete and accurate
- [ ] Featured image is present and relevant
- [ ] Tone matches the post type (conversational for tutorials, celebratory for client work, exploratory for game dev)
- [ ] No obvious typos or grammatical errors
- [ ] Bullet points and lists are formatted consistently
- [ ] Disclaimers are in place for your own modules/work
