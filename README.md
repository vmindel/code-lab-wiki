# code-lab-wiki

Public wiki of bioinformatics analyses, pipelines, scripts, and lab notes.
Source for [vmindel.github.io/code-lab-wiki](https://vmindel.github.io/code-lab-wiki/).

Built with [MkDocs](https://www.mkdocs.org/) + [Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages via Actions.

## Add a page

1. Create a Markdown file under `docs/`, e.g. `docs/bioinformatics/star-rnaseq.md`.
2. Optionally add it to the `nav:` block in `mkdocs.yml` (otherwise MkDocs picks it up automatically if `nav` is removed).
3. Commit and push to `main` — GitHub Actions rebuilds and republishes the site.

For the rendering features available (callouts, tabs, code blocks, tables), see `docs/bioinformatics/index.md` as a template.

## Preview locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>. Pages live-reload on save.

## First-time setup (once per repo)

After the first push to GitHub:

1. Repo **Settings → Pages → Build and deployment → Source: GitHub Actions**.
2. Re-run the failed first workflow if it ran before Pages was enabled.

The site URL will appear under Settings → Pages once the first deploy succeeds.
