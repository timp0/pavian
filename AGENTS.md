# Notes for future maintainers (human or AI)

This file is about **deploying the Shiny app to Posit Connect Cloud**. Nothing here
affects the R package itself — if you're just installing pavian with
`remotes::install_github()`, ignore all of this.

## The short version

The live app is deployed from this repo:

- **Repository:** `timp0/pavian`, branch `master`
- **Primary file:** `inst/shinyapp/app.R`
- **Dependency spec:** `inst/shinyapp/manifest.json`

Connect Cloud reads `manifest.json` to decide which R version and which packages to
install. It does **not** use renv.lock, DESCRIPTION, or packrat. If a deploy fails, the
manifest is almost always the thing to look at.

## How to regenerate manifest.json (the easy, correct way)

Do this on a machine with R 4.5 and the packages actually installed:

```r
install.packages(c("rsconnect", "remotes", "BiocManager"))
remotes::install_github(c("fbreitwieser/pavian",
                          "fbreitwieser/sankeyD3",
                          "fbreitwieser/shinyFileTree"))
BiocManager::install("Rsamtools")
setRepositories()  # tick CRAN + BioC software, so Bioc deps get recorded
rsconnect::writeManifest(appDir = "inst/shinyapp")
```

`writeManifest()` reads the DESCRIPTION of each **installed** package, which matters a
lot — see the third gotcha below.

## The four things that broke, and why

The app was originally on shinyapps.io. Moving it to Connect Cloud in September 2026
took four separate fixes. All of them are the kind of thing you lose an afternoon to, so
they're written out here.

### 1. Old package pins rot. CRAN deletes old versions.

The manifest exported from shinyapps.io was a packrat snapshot from **March 2021** — R
4.0.4, ~100 packages pinned at 2021 versions, all pointing at `https://cran.rstudio.com/`.

CRAN only serves the *current* version of each package from `src/contrib`. Everything
older gets moved to `src/contrib/Archive/`. So five years later, essentially every pinned
URL was a 404:

```
https://cran.rstudio.com/src/contrib/shiny_1.6.0.tar.gz    -> 404
https://cran.rstudio.com/src/contrib/rlang_0.4.10.tar.gz   -> 404
```

In the build log that shows up as `Could not download uncached package`.

**Fix:** regenerate with current versions, and point CRAN at a *dated* Posit Package
Manager snapshot rather than bare CRAN:

```
"Repository": "https://packagemanager.posit.co/cran/2026-09-05"
```

A dated snapshot keeps serving exactly those versions indefinitely, so the manifest
won't rot the same way. (Bare `cran.rstudio.com` or `cloud.r-project.org` will.)

Worth knowing: trying to *preserve* the 2021 environment by repointing it at a 2021
snapshot does **not** work. That lockfile was internally inconsistent — it held
`utf8 1.1.4` (2018) alongside `pillar 1.5.1` (2021), and no single snapshot date serves
both. Rebuilding is the only real option.

### 2. Connect Cloud adds `src/contrib` for you. Don't include it.

The `Repository` field must be the repository **root**. Connect Cloud appends
`src/contrib/<pkg>_<version>.tar.gz` itself. The shinyapps.io manifest had the full path
baked in for Bioconductor, which produced this:

```
https://bioconductor.org/packages/3.22/bioc/src/contrib/src/contrib/BiocParallel_1.44.0.tar.gz
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^ doubled -> 404
```

**Correct:**

```json
"Repository": "https://bioconductor.org/packages/3.22/bioc"
```

Note that shinyapps.io tolerated the doubled form, so a manifest that worked there can
still fail here.

### 3. GitHub packages need `Remote*` fields *inside* `description`. ← the sneaky one

pavian pulls three packages from GitHub: itself, `sankeyD3`, and `shinyFileTree`. For
those, the top-level `GithubRepo` / `GithubUsername` / `GithubSha1` keys are **not
enough**. renv resolves the download from fields that live inside the package's
`description` object:

```json
"shinyFileTree": {
  "Source": "github",
  "Repository": null,
  "GithubRepo": "shinyFileTree",
  "GithubUsername": "fbreitwieser",
  "GithubRef": "HEAD",
  "GithubSha1": "76c44e744c930c3908a98d4a055a5bff7fba1e0c",
  "description": {
    "Package": "shinyFileTree",
    "...": "...",
    "RemoteType": "github",
    "RemoteHost": "api.github.com",
    "RemoteRepo": "shinyFileTree",
    "RemoteUsername": "fbreitwieser",
    "RemoteRef": "HEAD",
    "RemoteSha": "76c44e744c930c3908a98d4a055a5bff7fba1e0c"
  }
}
```

Where do those `Remote*` fields come from? `remotes::install_github()` writes them into
the DESCRIPTION of the package when it installs it. `writeManifest()` then copies that
installed DESCRIPTION verbatim. So if you use the normal workflow they appear for free.

**The trap:** if you ever hand-build a manifest by scraping DESCRIPTION files from GitHub
(reasonable-looking shortcut when you don't have R handy), those files are the *source*
DESCRIPTIONs and carry none of the `Remote*` fields. Everything looks right, and the
deploy fails with a bare `Download of the shinyFileTree package has failed` and no URL to
tell you why. That cost three deploy cycles to find.

Also: don't add a `GithubHost` key with a scheme (`https://api.github.com`). Either omit
it or use the bare host. Same doubling hazard as #2.

### 4. Don't install packages at runtime

`inst/shinyapp/app.R` opens with `if (!require(pavian)) { ... install_github(...) }`.
That's fine locally and it's a no-op on Connect Cloud *provided the manifest did its
job* — `require()` succeeds and the install branch never runs.

But it can't rescue you. The deployed container's library is read-only, so if a package
is genuinely missing the install will fail too, just later and more confusingly. Fix the
manifest; don't lean on the runtime fallback.

## Operational notes

- **Build takes ~3.5 minutes**, most of it compiling htslib for `Rhtslib` from source.
  4 GB of memory is enough. If you see an OOM kill rather than a compile error, that's
  where it'll be.
- **Pinned sha:** the manifest installs pavian from `timp0/pavian` at a specific commit.
  Changing R code in `R/` does **not** reach the deployment until you bump `GithubSha1`
  and `description.RemoteSha` in the manifest. This is easy to forget and looks like
  "my change didn't deploy".
- **Auto-publish on push** is enabled and works, but it only started firing once the
  content had published successfully at least once. While the deploy was still failing,
  every retry needed the Republish button in the Connect Cloud UI. So if you're debugging
  a broken manifest, expect to click Republish rather than waiting on the webhook.
- **Stopping the app:** there is no stop or pause button, and you don't need one. Workers
  start on demand and shut down after the idle timeout (Settings -> Runtime, currently
  5 seconds), so nothing is running when nobody is connected. The only way to take it
  offline for good is the Delete item in the content's `...` menu, which is permanent.
- **Ephemeral filesystem:** `enableBookmarking = "server"` and pavian's
  `rappdirs::user_config_dir()` both write to the container, which is wiped on restart.
  Bookmarks won't survive.
- **Reading the logs:** Connect Cloud prints the failing URL for CRAN/Bioconductor
  problems, which makes those easy. It does *not* print a URL for GitHub package
  failures — for those, compare your manifest entry against the shape in gotcha #3.

## Version pairing

R and Bioconductor releases are locked together; you can't mix freely.

| R | Bioconductor |
|---|---|
| 4.5.x | 3.22 |
| 4.6.0 | 3.23 |

Current deployment: **R 4.5.1 / Bioconductor 3.22**, 93 packages. Connect Cloud supports
R 4.0.0 through 4.6.0. The authoritative mapping is at https://bioconductor.org/config.yaml
(`r_version_associated_with_release`).
