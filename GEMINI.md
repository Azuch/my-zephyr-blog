# Project Overview

This is a personal blog built with Jekyll, a static site generator. It uses the default "minima" theme. The blog is in its early stages, with most of the content being the default boilerplate from Jekyll.

## Building and Running

To work on this blog locally, you'll need to have Ruby and Jekyll installed.

1.  **Install dependencies:**
    ```bash
    bundle install
    ```

2.  **Run the local development server:**
    ```bash
    bundle exec jekyll serve --livereload
    ```
    This will start a local server, typically at `http://127.0.0.1:4000/`. The `--livereload` flag will automatically refresh the page when you make changes to the files.

## Development Conventions

*   **Content:** Blog posts are written in Markdown and stored in the `_posts` directory. The filename format for posts is `YYYY-MM-DD-title.markdown`.
*   **Customization:** The site's configuration can be modified in `_config.yml`. To customize the theme (e.g., layout, styles), you can override the theme's files. More information can be found in the [Jekyll documentation](https://jekyllrb.com/docs/themes/#overriding-theme-defaults).
*   **Pages:** New pages can be created by adding Markdown files to the root directory with the appropriate "front matter" (the block at the top of the file between the `---` lines).

---
*Managed by Gemini-CLI. Push access granted.*

---

## Project Log

This log summarizes the key development steps taken for the blog.

### 1. Initial Setup & Deployment
*   **Git Initialization:** Connected the local repository to the GitHub remote `Azuch/my-zephyr-blog` and configured it to use a PAT for authentication. Set local Git user to `azuch` (`namlucius@gmail.com`).
*   **Vercel Configuration:** Prepared the project for Vercel deployment by creating a `Gemfile` (with `jekyll` and `minima` gems) and adding `_site` and `*.log` to `.gitignore`.
*   **Deployment URL:** The live site is deployed at `https://my-zephyr-blog.vercel.app/`.

### 2. Content & Configuration
*   **Core Files:** Created the main configuration file `_config.yml` with the site title, author, and description.
*   **First Post:** Wrote and published the first blog post, "[Chào Zephyr, Tạm Biệt Thế Giới!](_posts/2025-12-04-chao-zephyr-tam-biet-the-gioi.markdown)".
*   **About Page:** Added an `about.md` page and linked it in the site's main header navigation.

### 3. Troubleshooting
*   **Fixed 404 Error:** Diagnosed and fixed a 404 error on the live site by creating a root `index.md` to serve as the homepage and list blog posts.
*   **Analyzed Build Warnings:** Investigated Vercel build logs and determined that numerous `DEPRECATION WARNING` messages were non-critical warnings originating from the `minima` theme's CSS, not build errors.

### 4. Current Status
*   The user has requested to change the blog's theme to match the style of `jaredwolff.com`.
*   Initial analysis showed the target site uses a Hugo theme named "Serif".
*   A proposal to build a custom theme from scratch was rejected by the user in favor of a simpler approach.
*   **Next Step:** Awaiting user approval to find a pre-existing Jekyll theme that resembles the target style.
