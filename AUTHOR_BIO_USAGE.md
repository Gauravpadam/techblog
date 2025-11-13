# Author Bio Usage Guide

## Automatic Display (Recommended)

By default, the author bio appears at the end of every blog post automatically. No action needed!

**What it shows:**
- Author name: Gaurav Padam
- Description: "I read code, write code and make random arrow diagrams to explain how my applications are supposed to work. I do some math as well."
- LinkedIn link

## Disable on Specific Posts

To hide the author bio on a specific post, add this to the post's front matter:

```yaml
---
title: "My Post Title"
date: 2025-11-14
hide_author_bio: true
---
```

## Manual Shortcode (Alternative)

If you prefer manual control, you can:

1. Disable automatic display by removing the partial from `single.html`
2. Add the shortcode manually in your posts:

```markdown
Your post content here...

{{< author >}}
```

## Configuration

Edit these values in `hugo.toml`:

```toml
[params]
  authors_name = 'Gaurav Padam'
  authors_description = "Your bio here"
  authors_linkedin = "https://www.linkedin.com/in/your-profile/"
```

## Styling

The author bio uses these CSS classes (in `main.css`):
- `.author-bio` - Main container
- `.author-bio-header` - Header section
- `.author-bio-content` - Content section

Customize the styling by editing these classes in `themes/techblog/static/css/main.css`.
