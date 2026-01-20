# Airbot Documentation Site

**Internal Documentation** - This is a Jekyll-based documentation site using the [Just the Docs](https://just-the-docs.github.io/just-the-docs/) theme, optimized for GitHub Pages.

This documentation is for internal team use only and includes search engine blocking via `robots.txt` and meta tags.

## Features

- **Sidebar Navigation**: Automatic sidebar with all documentation pages
- **Search**: Built-in search functionality
- **Responsive Design**: Mobile-friendly documentation
- **Code Highlighting**: Syntax highlighting for code blocks
- **Custom Dark Theme**: Optimized dark color scheme for better readability and reduced eye strain

## Setup Instructions

### 1. Update Configuration

Edit `_config.yml` and update these values:

```yaml
baseurl: "/your-repo-name"  # Replace with your repository name
url: "https://yourusername.github.io"  # Replace with your GitHub Pages URL
```

In the `aux_links` section, update the GitHub repository URL:

```yaml
aux_links:
  "View on GitHub":
    - "https://github.com/yourusername/your-repo-name"
```

### 2. Keep Repository Private (Recommended)

For internal documentation, consider keeping your GitHub repository private:

1. Go to your repository **Settings** → **General**
2. Scroll to the **Danger Zone**
3. Click **Change visibility** → **Make private**

**Note**: GitHub Pages on private repositories requires GitHub Pro, Team, or Enterprise. If using a free account, the site will be public even if the repository is private.

### 3. Enable GitHub Pages

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Pages**
3. Under **Source**, select:
   - Branch: `main` (or your default branch)
   - Folder: `/docs`
4. Click **Save**

GitHub will automatically build and deploy your site. It will be available at:
`https://lumunate.github.io/airbot-documentation/`

**Privacy Note**:
- `robots.txt` and meta tags are configured to prevent search engine indexing
- For true privacy, use a private repository with GitHub Pro/Team/Enterprise
- Alternatively, share the URL only with team members who need access

### 3. Local Development (Optional)

To test the site locally before deploying:

**Prerequisites:**
- Ruby (version 2.7 or higher)
- Bundler

**Steps:**

```bash
# Navigate to the docs directory
cd docs

# Install dependencies
bundle install

# Serve the site locally
bundle exec jekyll serve

# Open browser to http://localhost:4000
```

## File Structure

```
docs/
├── _config.yml              # Jekyll configuration
├── Gemfile                  # Ruby dependencies
├── README.md                # This file
└── docs/
    ├── index.md             # Home page (nav_order: 1)
    ├── 01-onboarding-and-setup.md
    ├── 02-access-control.md
    ├── 03-ai-features.md
    ├── 04-upsell-system-analysis.md
    ├── 05-business-logic.md
    └── 06-feature-implementation-analysis.md
```

## Customization

### Adding New Pages

To add a new documentation page:

1. Create a new `.md` file in the `docs/` directory
2. Add front matter at the top:

```yaml
---
layout: default
title: Your Page Title
nav_order: 8  # Determines sidebar position
description: "Brief description"
---
```

3. The page will automatically appear in the sidebar navigation

### Changing Theme

The site uses a custom dark theme optimized for Airbot documentation. To switch themes, edit `_config.yml`:

```yaml
# Custom Airbot dark theme (current)
color_scheme: airbot-dark

# Or use built-in themes:
# color_scheme: light
# color_scheme: dark
```

**Custom Theme Files:**
- Color scheme: `_sass/color_schemes/airbot-dark.scss`
- Additional styles: `assets/css/custom.scss`

### Adding a Logo

1. Add your logo image to `assets/images/logo.png`
2. Uncomment this line in `_config.yml`:

```yaml
logo: "/assets/images/logo.png"
```

## Theme Documentation

For more customization options, see the [Just the Docs documentation](https://just-the-docs.github.io/just-the-docs/).

## Troubleshooting

### Site not loading on GitHub Pages

- Ensure GitHub Pages is enabled in repository settings
- Check that the source is set to the correct branch and `/docs` folder
- Verify `baseurl` and `url` in `_config.yml` are correct
- GitHub Pages can take a few minutes to build and deploy changes

### Navigation not showing

- Ensure all `.md` files have proper front matter with `nav_order`
- Check that files are in the `docs/` subdirectory

### Search not working

- Search is automatically enabled for GitHub Pages
- It may take a few minutes after deployment for search index to build

## License

Documentation content is proprietary to Airbot.
