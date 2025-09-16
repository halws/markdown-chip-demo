# Deployment Guide

This project uses GitHub Actions for continuous integration and deployment. Production builds are automatically deployed to GitHub Pages.

## 🚀 Deployment Setup

### 1. Enable GitHub Pages

1. Go to your repository **Settings** → **Pages**
2. Under **Source**, select **GitHub Actions**
3. Save the settings

### 2. Repository Permissions

Ensure your repository has the following permissions for GitHub Actions:

- **Settings** → **Actions** → **General**
- **Workflow permissions**: Choose "Read and write permissions"
- **Allow GitHub Actions to create and approve pull requests**: ✅ Checked

## 🔄 Deployment Workflow

### Production Deployment

- **Trigger**: Push to `main` or `master` branch
- **Target**: GitHub Pages
- **URL**: `https://[username].github.io/textarea-overlay/`

## 🏗️ Build Process

The CI/CD pipeline includes:

1. **Build & Test**

   - Runs on Node.js 20 & 22
   - Type checking with TypeScript
   - Application build
   - Artifact upload

2. **Production Deploy**

   - Build application
   - Deploy to GitHub Pages
   - Environment: `github-pages`

## 📝 Testing Checklist

When testing deployments, verify:

- ✅ Hover functionality on note mentions
- ✅ Markdown rendering in readonly mode
- ✅ Textarea input and synchronization
- ✅ Debug mode toggle
- ✅ All assets load correctly
- ✅ Proper routing (if applicable)

## 🛠️ Local Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

## 🔧 Configuration

### Vite Configuration

- **Base Path**: `/textarea-overlay/` for production (GitHub Pages)
- **Base Path**: `/` for development
- **Environment**: Automatic detection via `NODE_ENV`

### Build Output

- **Directory**: `dist/`
- **Format**: Static files ready for deployment
- **Assets**: Optimized and hashed for caching
