# Deployment Guide

## Pushing to GitHub

### Initial Setup

1. **Initialize Git repository** (if not already initialized):
```bash
git init
```

2. **Add remote repository**:
```bash
git remote add origin https://github.com/Popdogos/Popdog.git
```

3. **Add all files**:
```bash
git add .
```

4. **Commit changes**:
```bash
git commit -m "Initial commit: Popdog website with wallet integration"
```

5. **Push to GitHub**:
```bash
git branch -M main
git push -u origin main
```

### Updating the Repository

After making changes:

```bash
git add .
git commit -m "Description of your changes"
git push origin main
```

## GitHub Pages Deployment

1. Go to repository Settings
2. Navigate to Pages section
3. Select source branch: `main`
4. Select folder: `/ (root)`
5. Click Save
6. Your site will be available at: `https://popdogos.github.io/Popdog/`

## Alternative Hosting Options

### Netlify
1. Drag and drop the `popdog` folder to Netlify
2. Or connect GitHub repository for automatic deployments

### Vercel
1. Import GitHub repository
2. Vercel will auto-detect settings
3. Deploy with one click

### Cloudflare Pages
1. Connect GitHub repository
2. Build command: (leave empty)
3. Build output directory: `/`
4. Deploy

## Environment Variables

No environment variables required for basic functionality.

## HTTPS Requirement

**Important**: Wallet connections require HTTPS. Ensure your hosting provider supports HTTPS (all major providers do by default).

## Custom Domain

To use a custom domain:
1. Add CNAME record pointing to your hosting provider
2. Configure domain in hosting provider settings
3. SSL certificate will be auto-generated

---

For questions or issues, please open an issue on GitHub.

