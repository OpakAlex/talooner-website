# talooner-website

Marketing and docs site for [Talooner](https://github.com/opentalon/talooner) — a
deterministic PR review bot. Built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to GitHub
Pages.

## Sections

- **How it works** — the end-to-end flow and where a model fits (`content/how-it-works.md`)
- **Automated QA** — rules that decide to deploy and verify with LLM-generated scenarios (`content/automated-qa.md`)
- **Rules** — policy as code (`content/rules.md`)
- **Blog** — `content/blog/`

## Local development

The PaperMod theme is a git submodule.

```bash
git clone --recurse-submodules <repo-url>
# or, if already cloned:
git submodule update --init --recursive

hugo server -D          # preview at http://localhost:1313
hugo new blog/my-post.md  # scaffold a new post from archetypes/blog.md
```

## Deploy

Pushing to `master` triggers `.github/workflows/deploy.yml`, which builds with Hugo
extended and publishes to GitHub Pages.
