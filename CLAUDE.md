# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo static site generator (SSG) for a personal blog/portfolio at andywhale.co.uk. Hugo transforms markdown posts and configuration into a static HTML site.

## Architecture

**Hugo Site Structure:**
- `content/posts/` — All blog post markdown files. Each post is a standalone `.md` file with front matter (metadata) followed by content.
- `themes/mediumish/` — The main visual theme (configured as a git submodule in `.gitmodules`). Theme contains templates, CSS, and layout logic.
- `layouts/` — Custom layout overrides that extend/replace theme templates. Currently contains:
  - `index.html` — Custom homepage layout that renders the index section from `config.toml` parameters
  - `partials/` — Reusable template partials (navbar, social links, etc.)
- `static/images/` — Image assets referenced in posts via the `image` front matter field
- `config.toml` — Hugo configuration: site title, base URL, theme, menus, author info, and index page content
- `public/` — Generated HTML output (ignored in git, created by Hugo build)

**Post Structure:**
Posts use Hugo front matter (YAML between `---` markers) with required/optional fields:
- `title` — Post title
- `description` — Short summary (used in meta tags and listings)
- `date` — Publication date (ISO 8601 format)
- `image` — Featured image path (e.g., `images/drupal-accessible-media.png`)
- `type` — Always `"post"`
- `tags` — Array of tags for categorization (e.g., `["drupal", "php", "accessibility"]`)
- `draft` — Boolean to hide unpublished posts (defaults to `false` for live posts)

The post content follows the front matter and uses standard markdown. Posts can include:
- Inline links: `[text](url)`
- Headings: `## Section Title`
- Lists: Bullet points and numbered lists
- Embedded content: Hugo shortcodes like `{{< youtube VIDEO_ID >}}`
- HTML (e.g., embedded tweets via `<blockquote class="twitter-tweet">...</blockquote>`)

## Common Commands

**Development:**
```bash
# Start local development server (watches for changes, rebuilds automatically)
hugo server

# Or with draft posts visible:
hugo server -D
```

**Building:**
```bash
# Generate the static site (outputs to public/)
hugo

# Build with drafts included:
hugo -D
```

**Creating new posts:**
```bash
# Create a new post with the archetype template
hugo new content/posts/my-post-title.md

# This generates a file with front matter and default values from archetypes/default.md
```

**Preview in browser:**
Once `hugo server` is running, the site is available at `http://localhost:1313` and automatically rebuilds as you edit markdown or config files.

## Writing & Post Guidelines

See the repository's writing style documented elsewhere for tone and content patterns. Key points for Hugo posts:
- Keep `description` concise (used in meta tags and post listings)
- Use descriptive `tags` to enable content discovery
- Include an `image` for visual appeal in post listings
- Set `draft: false` only when ready to publish
- Link to external resources generously (Drupal modules, external sites, etc.)

## Theme & Customization

The `mediumish` theme handles most styling and post rendering. Custom overrides in `layouts/` take precedence over theme files:
- Modify `layouts/index.html` to change the homepage layout or the sections displayed (pulls from `config.toml` parameters like `index.mdtext`, `index.mdtext2`, `index.mdtext3`)
- Modify `layouts/partials/` to customize navbar, social links, or other shared elements
- For deeper theme changes, edit files in `themes/mediumish/` or override them in `layouts/`

The site uses social links configured in `config.toml` under `[params.social]` (GitHub, LinkedIn, Twitter, Instagram, Drupal, Flickr).

## Deployment

The `public/` directory contains the generated static site. This is typically deployed to a web server or CDN. The site's baseURL is configured in `config.toml` as `https://andywhale.co.uk`.
