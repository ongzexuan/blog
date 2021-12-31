# blog

GitHub pages hosted blog for myself. This repo contains the source files.

## Setup (for when I forget my workflow)

The page is built using the [Hugo framework](https://gohugo.io/). This can be installed using Homebrew. This provides the `hugo` command, which will take the source files in this repo and generate a static html page that can be hosted on GitHub.

### Building the static site

Running the `hugo` command will generate the static site in a directory `ongzexuan.github.io`. By default, the directory is named `public`, however we amend the `publishDir` config option in `config.toml` to rename the directory. We do this because I've added `ongzexuan.github.io` as a submodule to this repo (that repo holds the static page only, which GitHub serves).

To simplify this workflow, there is a deploy script `deploy.sh`, which when run with a string argument, will build the page, and then git add commit push the page to the static repo (the string is the commit message).

### Navigating this repo

The main content resides in the `content` directory, under `posts`. The static content (images, etc.) goes under `static`. For better organization, the practice here is to create a subdirectory in `static` for each respective markdown file (post) in `posts` (if there is any static content to serve).