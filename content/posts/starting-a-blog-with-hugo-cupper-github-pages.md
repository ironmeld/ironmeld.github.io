---
title: "Starting a Blog with Hugo, Cupper, and GitHub Pages"
date: 2022-02-15T12:00:00Z
draft: false
tags: ["blog", "hugo", "cupper", "github"]
toc: true
---

## Introduction

This blog post explains how this blog was created. It is an integration of:
* The [Hugo Quick Start](https://gohugo.io/getting-started/quick-start/)
* The [Cupper Theme README](https://github.com/zwbetz-gh/cupper-hugo-theme)
* Hugo's [Hosting on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/) instructions.

I wrote this blog post because I could not find any step-by-step instructions that actually work. For example, almost all start with "hugo new site ..." but then don't explain how to get that new directory into git or GitHub. It's an exercise left to the reader. The standard instructions suggest using "git submodule add" command to import a theme, but that only works if your directory is already in git.

Piecing together the correct steps is not trivial. But getting started quickly with Hugo should be quick and easy. It is Hugo's strongest advantage, in my opinion. So I wrote these instructions.

## Assumptions

This post is not intended to advocate for Hugo. I assume you have done your research and considered the alternatives. A popular alternative is Jekyll, which is the default tool used by GitHub Pages.

Factors in *my* decision:

* Installation: Hugo consists of one binary
* Maintenance: Hugo is not [hard to maintain](https://news.ycombinator.com/item?id=12675391). Life is too short to waste it [babysitting](https://news.ycombinator.com/item?id=12673949) fragile development ecosystems.
* Stablility: GitHub managing Jekyll means GitHub can change how it works at any time and [break your site](https://joearms.github.io/#2016-05-31%20New%20Blogging%20Software).
* Speed: Hugo runs fast
* Beauty: I found a [theme](https://themes.gohugo.io/) that I liked
* Language: I prefer Golang over Ruby (Jekyll is written in Ruby).

## Prerequisites

1. You have a development environment with bash and you do not need hand holding
2. You know how to use GitHub and git
3. You are in a directory named \<USERNAME\>.github.io containing each of your blog posts as a separate markdown file
4. Each post has a header with the following format:
```yaml
---
title: "Starting a Blog with Hugo, Cupper, and GitHub Page"
date: 2022-02-15T12:00:00Z
draft: false
tags: ["blog", "hugo", "cupper", "github"]
toc: true
---
```
5. The post title will be displayed; you do not need to repeat it. Your first header should start with two pound signs. 


## Theory of Operation

This is an overview of the process. Details are provided in the rest of the document.

* You will check your markdown documents into the `main` branch using the Hugo directory structure.
* Any time you commit changes to your content on the `main` branch, a GitHub Action (which you will setup) is triggered to run hugo and produce the static html pages.
* The GitHub Action then automatically checks the html pages into the `gh-pages` branch.
* A separate built-in GitHub Action then publishes any changes to `gh-pages` onto the internet.

## Create the GitHub repo
First, go to the GitHub web UI and create a new repo named \<USERNAME\>.github.io and check the box to create a README.md. This creates a `main` branch.

In the Description, create a link to the published GitHub Pages (**replacing USERNAME with your own**):
```markdown
Source for the blog [USERNAME.github.io](https://USERNAME.github.io).
```

{{< note >}}
Your initial README.md will be published as a GitHub Page immediately using Jekyll to build the HTML. This make take a minute. Check the Actions tab for status. You can now test that your blog is reachable on the Internet. In the remaining instructions we will turn off Jekyll and switch to Hugo.
{{< /note >}}


## Configure GitHub Pages to Publish from the `gh-pages` Branch
Go to the Code of the repository, click on **main** dropdown, type `gh-pages` and then click on "Create branch: gh-pages from main".

On the repository page in GitHub, go to Settings, click Pages on the left tab menu. Change the Source to `gh-pages`. Click Save.


## Create the Hugo config.yml template

Place this in a file name config.yml:

```yaml
baseURL: https://REPONAME
languageCode: en-us
title: The Blog for USERORG

theme: cupper-hugo-theme

taxonomies:
  tag: tags

permalinks:
  post: /:filename/

menu:
  nav:
    - name: Home
      url: /
      weight: 1
    - name: Blog Posts
      url: /posts/
      weight: 2
    - name: Tags
      url: /tags/
      weight: 3
    - name: RSS
      url: /index.xml
      weight: 4

params:
  footer: Made with [Hugo](https://gohugo.io/). Themed by [Cupper](https://github.com/zwbetz-gh/cupper-hugo-theme).
  # For more date formats see https://gohugo.io/functions/format/
  dateFormat: Jan 2, 2006
  katex: true
  hideHeaderLinks: false
  search: true
  showThemeSwitcher: true
  defaultDarkTheme: false
  moveFooterToHeader: false
  logoAlt: Logo

markup:
  tableOfContents:
    endLevel: 6
    startLevel: 2
```

## Create GitHub Action file gh-pages.yml

Place the following contents into a file named gh-pages.yml.

```yaml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

## Install Hugo

Get the Hugo binary for your platform from the [Hugo releases page](https://github.com/gohugoio/hugo/releases) and place it in your PATH.

## Run makeblog.sh script

Then you can get published by running the bash script listed in the next section. Copy and paste the contents into a file named makeblog.sh in the current directory and be sure to examine it thoroughly so that you understand what it is doing. Make it executable and run it. It may prompt for your Github credentials if you have it configured that way. When it is done, your hugo site should be up and running!

```bash
$ pwd
/home/rick/projects/rick-masters.github.io
$ ls -1
config.yml
gh-pages.yml
makeblog.sh
starting-a-blog-with-hugo-cupper-github-pages.md
$ ./makeblog.sh
```

## Contents of the makeblog.sh script

```bash
#!/bin/bash
# -e abort on errors
# -u undefined vars are errors
# -o pipefail: any command can fail a pipeline
set -euo pipefail

# Get repo name from the name of the current directory
REPONAME=`basename "$PWD"`

# Strip off .github.io suffix to get the user name
USERORG="${REPONAME%.github.io}"

# create .git directory
git init .
git remote add origin "git@github.com:/$USERORG/$REPONAME"
# don't worry; this does not wipe your documents in the current directory
git pull origin main
git checkout main

# Tell GitHub not to bother with Jekyll
touch .nojekyll

# Setup GitHub Action to build web with Hugo
mkdir -p .github/workflows
mv gh-pages.yml .github/workflows

# --force ignores existing directory 
hugo new site . --force
git submodule add https://github.com/zwbetz-gh/cupper-hugo-theme.git themes/cupper-hugo-theme

# customize the hugo configuration
sed -i "s/REPONAME/$REPONAME/" config.yml
sed -i "s/USERORG/$USERORG/" config.yml

# remove the stock config
rm config.toml

# populate the home page
cat > content/_index.md << HOMEEND
---
title: "My Blog"
---
This is my blog.

To see the list of blog posts click [here](/posts "Blog Posts") or use the menu.
HOMEEND

# Move posts into position
mkdir content/posts
for post in *.md
do
    [ "$post" = "README.md" ] && continue
    mv "$post" content/posts/
done

# Set the title for the list of posts
printf -- "---\ntitle: Blog Posts\n---\n" > content/posts/_index.md

# Format for a post
mkdir layouts/_default
cp themes/cupper-hugo-theme/layouts/post/single.html layouts/_default/

# commit and push
git add .
git reset makeblog.sh  # don't include this script
git commit -m "Initial hugo blog"
git push -u origin main
```

## Adding a custom logo

While not required, you should probably change the logo from the default "cupper" coffee cup logo in order to establish your own brand and to avoid questions about why your blog is called "cupper".

Create a directory named `static/images` and place a file named `logo.svg` in that directory. Push to GitHub. If you need a tool for creating an SVG file, try [Boxy](https://boxy-svg.com/) or [Inkscape](https://inkscape.org/).

## Adding images

Place your images in `static/images/`. Create your links using `/images/` like so:

```markdown
![My Image Text](/images/imagename.png)
```

## Reversing the list of posts

By default, newer posts are listed above older posts. If you want to reverse the order, you will need to create a custom list.html.

{{< note >}}
This step is optional.
{{< /note >}}


```bash
cp themes/cupper-hugo-theme/layouts/_default/list.html layouts/_default/
```

Now, edit the file to customize the html. These are the edits I am using:

```bash
$ diff -u themes/cupper-hugo-theme/layouts/_default/list.html layouts/_default/list.html 
--- themes/cupper-hugo-theme/layouts/_default/list.html 2022-02-04 20:51:00.812560573 +0000
+++ layouts/_default/list.html  2022-02-05 15:35:15.770885365 +0000
@@ -10,7 +10,7 @@
   />
   {{ end }}
   <ul class="patterns-list" id="list">
-    {{ range .Pages.ByPublishDate.Reverse }}
+    {{ range .Pages.ByPublishDate }}
     <li>
       <h2>
         <a href="{{ .RelPermalink }}">
@@ -23,6 +23,11 @@
             <use xlink:href="#bookmark"></use>
           </svg>
           {{ .Title }}
+          {{ $dateFormat := $.Site.Params.dateFormat | default "Jan 2, 2006" }}
+          {{ $publishDate := .PublishDate }}
+          <span style="font-size: 0.80rem">
+          - {{ $publishDate.Format $dateFormat }}
+          </span>
         </a>
       </h2>
     </li>
```

This format also adds a date to each of the posts in the list. Push to GitHub and the list should now be sorted earliest to latest.

## Syntax Highlighting

Syntax highlighting did not work "out-of-the-box" for me. This is actually [documented](https://github.com/zwbetz-gh/cupper-hugo-theme#syntax-highlighting). Unfortunately, if you try to use a code block for any unsupported language or format, the code block will have *totally unreadable* colors. It seems like this could be handled better, but it is what it is.

So, you *MUST* go to https://prismjs.com/download.html and download a custom `prism.css` and `prism.js` file with your selected formats enabled. You will place those in the `static/css/` and the `static/js/` directories, respectively.

After I did that, it was working but I noticed my YAML strings were very dark. The reasons for this are:
1. The syntax highlighter thinks that unquoted YAML strings warrants a crappy color compared to the pleasant light green of quoted strings.
2. Prism supports different themes and the default theme is particularly unkind to unquoted YAML strings.

So, I selected the "Tomorrow Night" theme so that unquoted strings in YAML would look tolerable.


## Cloudflare Analytics

Hugo 0.92 supports Google Analytics but does not support Cloudflare analytics. Use these instructions to include the Cloudflare javascript instead.

1. Sign up for Cloudflare Analytics and copy the javascript snippet they provide
2. In your Hugo site, put the javascript in a file called `layouts/partials/cloudflare-analytics.html`, with the following format:
```html
{{ if not .Site.IsServer }}
<!-- put cloudflare analytics javascript here -->
{{ end }}
```

Then run these commands to switch the included partial from Google to Cloudflare.

```bash
cp themes/cupper-hugo-theme/layouts/_default/baseof.html layouts/_default/
sed -i 's/google-analytics-async/cloudflare-analytics/' layouts/_default/baseof.html
```

The End.
