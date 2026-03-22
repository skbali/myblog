# Blog by SK Bali

Personal blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Hosted on GitHub Pages.

## Tech Stack

- **Static site generator:** Hugo
- **Theme:** PaperMod
- **Hosting:** GitHub Pages
- **CI/CD:** GitHub Actions (auto-deploys on push to `main`)

## Project Structure

```
content/        # Blog posts and pages (Markdown)
layouts/        # Custom layout overrides
static/         # Static assets
themes/         # PaperMod theme (git submodule)
hugo.yaml       # Hugo configuration
.github/        # GitHub Actions workflow
```

## Local Development

**Prerequisites:** [Hugo extended](https://gohugo.io/installation/) must be installed.

```sh
# Clone with submodules (required for the theme)
git clone --recurse-submodules <repo-url>

# Start local dev server
hugo server -D
```

The site will be available at `http://localhost:1313`.

## Publishing

Push changes to the `main` branch (directly or via a pull request) to trigger an automatic build and deploy via GitHub Actions.
