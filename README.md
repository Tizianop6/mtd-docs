# Access at: https://cmsroma.github.io/mtd-docs/

# Installation
To edit, you should first install MkDocs and Material for MkDocs locally -- this is the most convenient way to locally test changes before pushing and also to deploy updates to Github Pages.

```bash
pip install mkdocs mkdocs-material python-markdown-math
```


# Local deployment

```bash
git clone git@github.com:CMSROMA/mtd-docs.git
cd mtd-docs
mkdocs serve --livereload
```

By default, this will deploy a local MkDocs server to `http://127.0.0.1:8000/`. The `--livereload` option ensures the page is instantly updated as you're editing files.

# Remote deployment

After having pushed your changes to the repo, you can also update the `gh-pages` that feeds the actual website by simply running inside the `mtd-docs` folder:

```bash
mkdocs gh-deploy
```

# Editing

## Folder structure
The overall structure tree of the website is defined in `mkdocs.yml` (see `nav` entry). e.g.:
```yml
nav:
  - Resources: 
    - Home: index.md
    - Samples: home/samples.md
    - Presentations: home/presentations.md
    - Useful links: home/links.md
  - TOF:
    - Modules:
      - TrackExtenderWithMTD: TOF/modules/TrackExtenderWithMTD.md
    
  - Clustering: 
    - Overview: clustering/overview.md
    - Modules: clustering/modules.md
    - RECO:
      - Subpage placeholder: clustering/reco/placeholder.md
    - SIM:
      - Subpage placeholder: clustering/sim/placeholder.md
```

This is relevant if you want to add subpages or change their name, for instance. For now it's mostly placeholders (plus one old page I wrote for TrackExtenderWithMTD last year, but it can be deleted).

## Pages
Every page is a .md file. They can be anywhere, but for tidiness I've put them in a `docs` subfolder with a folder substructure mirroring the website branches.

## Syntax
You can use Markdown syntax (e.g. https://www.mkdocs.org/user-guide/writing-your-docs/#writing-with-markdown), plus many additional frills from Material for Mkdocs (https://squidfunk.github.io/mkdocs-material/reference/).
