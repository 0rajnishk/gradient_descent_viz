# GitHub Pages Deployment

This directory contains the static website for the Gradient Descent Visualization project that is deployed to GitHub Pages.

## Files

- `index.html` - The main landing page for the project

## Deployment

The site is automatically deployed via GitHub Actions when changes are pushed to the `master` branch. The workflow is defined in `.github/workflows/deploy-pages.yml`.

## Local Testing

To test the site locally, simply open `index.html` in your web browser, or use a simple HTTP server:

```bash
# Using Python 3
python -m http.server 8000

# Using Python 2
python -m SimpleHTTPServer 8000

# Then visit http://localhost:8000 in your browser
```

## Customization

The landing page is a single HTML file with inline CSS. To customize:

1. Edit `docs/index.html`
2. Commit and push changes to master
3. The site will automatically redeploy via GitHub Actions

## Requirements

For the deployment to work, GitHub Pages must be enabled in the repository settings with source set to "GitHub Actions".
