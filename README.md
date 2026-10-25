# The documentation of Web Push

The public documentation of [Web Push](https://github.com/Spomky-Labs/web-push), web push notifications for
PHP and Symfony: the library and the Symfony bundle.

Published with [MkDocs](https://www.mkdocs.org/) and
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) on GitHub Pages:
**<https://web-push.spomky-labs.com/>**

## Writing

```bash
make install   # the toolchain, in .venv, from requirements.txt
make serve     # http://127.0.0.1:8000, rebuilt as you save
make check     # the strict build continuous integration runs
```

`make check` is the gate: a broken internal link, an anchor that does not exist, a page absent from the
navigation or a navigation entry without a page fails the build.

## Where things are

| Path | What it is |
|---|---|
| `mkdocs.yml` | The site: theme, extensions, plugins, and the whole navigation |
| `docs/` | The pages, one directory per section of the navigation |
| `docs/assets/stylesheets/` | The look: `slabs-tokens.css`, copied as it is from the Slabs design kit, and `slabs.css`, which applies the guidelines to the theme |
| `docs/assets/brand/`, `docs/assets/fonts/` | The logotype, the signature, the favicons and the share image of the design kit, and its typefaces (SIL Open Font License) |
| `docs/assets/images/` | The images of the pages |
| `docs/overrides/` | Theme partials: the logo, the footer, the link to the repository |
| `site-root/` | The root of the published site, copied to the root of the `gh-pages` branch: `CNAME`, and the `404.html` that sends the addresses of the former GitBook site to the latest version |
| `requirements.txt` | The toolchain, pinned. Continuous integration installs exactly this |
| `.github/workflows/check.yml` | The strict build, on every pull request |
| `.github/workflows/pages.yml` | The publication: one version per branch, through mike |
| `.github/workflows/default-branch.yml` | A new version branch becomes the default branch |

How the documentation left GitBook, and the rules every documentation site of Slabs follows, are in
[Spomky-Labs/docs](https://github.com/Spomky-Labs/docs/blob/main/MIGRATING.md).

## One branch per version

A branch of this repository is a version of the packages, named like their maintenance branches: `3.2.x`
publishes `3.2`. A push to one of them publishes it under its version, and the newest one also carries the
`latest` alias, which the root of the site redirects to.

```bash
git switch -c 3.3.x 3.2.x      # from the version it continues
# in mkdocs.yml, point edit_uri at the new branch
git push -u origin 3.3.x       # publishes it, and moves "latest" and the default branch
```

## Licence

The packages this documentation describes are released under the MIT licence, reproduced on the
[licence page](docs/license.md). This repository has no licence file of its own.
