# GitHub Copilot Instructions for The BS Blog

## Project Overview
This is a Jekyll-based technical blog hosted on GitHub Pages focused on development platform modernization, TypeScript learning journeys, and DevOps practices.

## Key Architecture Patterns

### Blog Structure
- **Posts Organization**: Content is organized in themed subdirectories under `_posts/`:
  - `typescriptjourney/` - Complete TypeScript learning series (9 posts, 2022)
  - `platform/` - Modern platform development series (started 2025)
- **Content Categories**: Use `categories` frontmatter for grouping (e.g., `["learning","typescript","Typescript Journey"]`)
- **Post Naming**: Follow Jekyll convention: `YYYY-MM-DD-title.md` for published posts
- **Draft Management**: Use `_drafts/` for unpublished content (no date in filename required)

### Jekyll Configuration
- **Theme**: Uses remote theme `slee981/jekyll-theme-cadre` (not standard minima)
- **Plugins**: GitHub Pages compatible plugins only (`jekyll-feed`, `jekyll-paginate`, `jekyll-seo-tag`, `jekyll-sitemap`)
- **Content Types**: 
  - Regular posts in `_posts/`
  - Static pages: `about.markdown`, archive/categories pages
  - Navigation via `_data/navigation.yml` and `_data/social.yml`

## Development Workflows

### Local Development
```bash
# Install dependencies
bundle install

# Serve locally (auto-regenerates)
bundle exec jekyll serve

# Build for production
bundle exec jekyll build
```

### Content Creation
- **New Posts**: Copy from `.post-template` for consistent frontmatter structure
- **Blog Post Headers**: Always include proper YAML frontmatter with title, date, draft status, categories, description, and layout
- **Series Posts**: Include series navigation link at top: `[Series Posts](https://brianpsheridan.com/categories.html#typescript-journey)`
- **Code Examples**: Use fenced code blocks with language specification
- **Images**: Store in `static/images/posts/` directory

## Content Conventions

### Frontmatter Patterns
Always include complete YAML frontmatter headers for blog posts. Use this structure:

```yaml
---
title: "Building a Modern Development Platform: From Legacy .NET Framework to Cloud-Native"
date: 2025-10-04T06:00:00-07:00
draft: false
categories: ["platform","aspire","typespec","kiota","dotnet","terraform","modernization","cloud"]
description: "A journey from legacy on-premise .NET Framework applications to a modern, cloud-native platform with standardized tooling, IaC, and developer experience"
layout: post  # Optional, defaults to post for _posts/ directory
---
```

**Required Fields:**
- `title`: Descriptive title (often includes emojis for visual appeal)
- `date`: ISO format with timezone (YYYY-MM-DDTHH:MM:SS-TZ)
- `draft`: Boolean for publication status
- `categories`: Array for content organization and grouping
- `description`: SEO-friendly meta description

### Writing Style
- **Emojis**: Widely used in titles and section headers for visual appeal
- **Code Structure**: Multi-step tutorials with clear goals, setup, and implementation
- **Cross-linking**: Link to related posts in series and GitHub source code
- **Testing Examples**: Include curl commands for API testing

### Emoji Usage Guidelines
Emojis significantly improve readability and visual scanning of blog posts. Use them strategically:

**Main Section Headers:**
- 📚 Documentation/Learning related
- 🏗️ Architecture/Structure
- ☁️ Cloud services setup
- 🔄 Workflows/Processes
- 🚀 Deployment
- ✅ Testing/Validation
- 🎯 Best practices/Goals
- 🔧 Configuration/Setup
- 💰 Cost/Pricing
- 🧹 Cleanup/Maintenance

**Inline Elements:**
- ✨ Features/Benefits
- ⚠️ Important warnings/caveats
- 💡 Tips/Insights
- 🔐 Security/Credentials
- 📊 Data/Statistics
- 🗂️ Structure/Organization
- 📋 Lists/Configuration
- 🌍 Global/Distributed
- 🗄️ Storage/Database
- 🆓 Free tier/No cost

**Callout Blocks:**
- ✅ Success/Working state
- ❌ Failures/Don't do this
- 🤔 Decision points
- 📌 Examples
- 🎨 Design elements

**Best Practices:**
1. Add emoji to main section headers (##) for visual hierarchy
2. Add emoji to subsection headers (###) when they have distinct purposes
3. Use inline emojis for callouts and lists to highlight important information
4. Keep emoji usage consistent within similar content types
5. Don't overuse - one emoji per section header is standard
6. Choose emojis that visually represent the content (not random)

### TypeScript Journey Series Patterns
- Each post builds on previous ones
- Include Docker/devcontainer setup
- VS Code extension recommendations
- Step-by-step code examples with file structure
- GitHub repo branches for each lesson

## File Organization
- **Assets**: Static files in `static/` (images, avatars, logos)
- **Data**: YAML configuration in `_data/` (navigation, social links)
- **Generated**: Avoid editing `_site/`, `Gemfile.lock` (auto-generated)

## GitHub Pages Deployment
- **Auto-deployment**: Pushes to `main` branch trigger GitHub Pages build
- **Domain**: Custom domain `brianpsheridan.com` via `CNAME` file
- **Theme Dependencies**: Uses `github-pages` gem for compatibility

## Common Tasks
- **New Tutorial Series**: Create subdirectory under `_posts/`, use consistent categories
- **Adding Images**: Upload to `static/images/posts/`, reference with `/static/` path
- **Navigation Updates**: Modify `_data/navigation.yml` for menu changes
- **Social Links**: Update `_data/social.yml` for footer social icons