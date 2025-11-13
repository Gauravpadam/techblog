# GitHub Pages Deployment Guide

This document provides instructions for deploying the tech blog to GitHub Pages using GitHub Actions.

## Prerequisites

- A GitHub repository containing this Hugo project
- GitHub Pages enabled for your repository

## Configuration Steps

### 1. Update Base URL

The `hugo.toml` file has been configured with a placeholder baseURL. Update it to match your GitHub Pages URL:

**For a user/organization site:**
```toml
baseURL = 'https://username.github.io/'
```

**For a project site (current configuration):**
```toml
baseURL = 'https://username.github.io/repository-name/'
```

Replace `username` with your GitHub username and `repository-name` with your repository name.

### 2. Configure GitHub Pages in Repository Settings

1. Go to your repository on GitHub
2. Click on **Settings** tab
3. Navigate to **Pages** in the left sidebar
4. Under **Build and deployment**:
   - **Source**: Select "GitHub Actions"
   - This allows the workflow to deploy directly without using a gh-pages branch

### 3. Push to Main Branch

Once configured, any push to the `main` branch will automatically trigger the deployment workflow:

```bash
git add .
git commit -m "Deploy blog"
git push origin main
```

### 4. Monitor Deployment

1. Go to the **Actions** tab in your repository
2. You'll see the "Deploy Hugo site to GitHub Pages" workflow running
3. Once complete (green checkmark), your site will be live at your configured baseURL

### 5. Access Your Site

After successful deployment, your blog will be available at:
- User/org site: `https://username.github.io/`
- Project site: `https://username.github.io/repository-name/`

## Workflow Details

The GitHub Actions workflow (`.github/workflows/deploy.yml`) performs the following steps:

1. **Checkout**: Clones the repository
2. **Setup Hugo**: Installs Hugo Extended v0.120.0
3. **Setup Pages**: Configures GitHub Pages settings
4. **Build**: Runs `hugo --minify` to generate the static site
5. **Upload**: Uploads the `public/` directory as an artifact
6. **Deploy**: Deploys the artifact to GitHub Pages

## Manual Deployment

You can also trigger deployment manually:

1. Go to the **Actions** tab
2. Select "Deploy Hugo site to GitHub Pages"
3. Click **Run workflow**
4. Select the branch and click **Run workflow**

## Troubleshooting

### Site Not Loading

- Verify the baseURL in `hugo.toml` matches your GitHub Pages URL exactly
- Check that GitHub Pages is enabled in repository settings
- Ensure the workflow completed successfully in the Actions tab

### 404 Errors on Subpages

- Confirm baseURL ends with a trailing slash
- Check that all internal links use Hugo's `relref` or `ref` functions

### Workflow Failures

- Check the Actions tab for error messages
- Verify the repository has proper permissions for GitHub Actions
- Ensure the `main` branch exists and contains the Hugo project

## Custom Domain (Optional)

To use a custom domain:

1. Add a `CNAME` file to the `static/` directory with your domain
2. Configure DNS settings with your domain provider
3. Update baseURL in `hugo.toml` to your custom domain
4. Enable "Enforce HTTPS" in GitHub Pages settings

## Local Testing

Before deploying, test your site locally:

```bash
# Development server with drafts
hugo server -D

# Production build
hugo --minify

# Test the production build
cd public && python3 -m http.server 8000
```

Visit `http://localhost:1313` (dev server) or `http://localhost:8000` (production build) to preview.
