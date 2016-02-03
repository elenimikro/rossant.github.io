Title: Setting up a blog with Pelican and GitHub Pages
Tags: python

I describe how I set up my static blog/website in Python with [Pelican](http://blog.getpelican.com/), [pandoc](http://pandoc.org/), [Docker](https://www.docker.com/), [Dockerhub](http://www.dockerhub.com/), [GitHub pages](https://pages.github.com/), and [Travis CI](https://travis-ci.org/).

<!-- PELICAN_END_SUMMARY -->

Here is the workflow I wanted to have:

1. I write new contents in Markdown files.
2. I commit and push the sources to my GitHub repository.
3. That's it. The website is automatically updated after two minutes, thanks to **Travis CI** and **Docker**. My CV is also automatically converted from Markdown to PDF via LaTeX.

Setting this up was not straightforward and it did require significant upfront investment.

## Creating a GitHub repo for your website

I assume you're creating your personal webpage. You need to create a `yourname.github.io` repo where `yourname` is your GitHub login. You can also use the same method for an organization page, with minor adjustments. [My repo is here](https://github.com/rossant/rossant.github.io/).

The `master` branch will contain the automatically-generated HTML contents. Only Travis CI will write to this branch with a force push. The sources will be in an [*orphan branch*](https://git-scm.com/docs/git-checkout/) named `sources`.

Clone your repo. Let's assume your local path is `/home/yourname/git/yourname.github.io/`.


## Setting up Pelican

Now create your website locally with Pelican. If you're starting from scratch this is not the simplest step! You can refer to [Pelican's documentation](http://docs.getpelican.com/en/stable/).

Here is an excerpt of my repo's file structure:

```
pelican-plugins/                    # git clone the official pelican-plugins repo
plugins/                            # put your own plugins here
content/                            # put your contents here
  images/                           # images that you use in your posts
  pages/                            # static pages
    about.md
    ...
  2016-01-01-my-blog-post.md        # URL will be http://yourname.github.io/my-blog-post/
  ...
themes/                             # put your themes and templates here
  pure/
    static/
    templates/
output/                             # HTML output generated by Pelican

Dockerfile                          # these two files are for Docker
run_docker.sh

Makefile                            # these files are auto-generated by Pelican
pelicanconf.py
publishconf.py
```

I write pages and posts in Markdown files within the `contents/` subdirectory. I can use the Jupyter Notebook to edit Markdown files with the [ipymd](http://github.com/rossant/ipymd) package. This is convenient when my posts contain a lot of code.

The theme's files (jinja templates, CSS and JS files) are in `themes/pure/` ([pure](http://purecss.io/) is the name of the CSS framework I'm using). I use a few Pelican plugins, which are in the `pelican-plugins/` subdirectory (a [cloned repo](https://github.com/getpelican/pelican-plugins)). I also have a custom plugin in `plugins/` (see later in this post).

When Pelican generates the website, the HTML files are saved in the `output/` subfolder which is *not* tracked by git.

Pelican comes with a tool that initializes the file structure for a new site. It creates a default `pelicanconf.py`, a complete `Makefile`, and a few other things. You put all of your site's parameters in `pelicanconf.py`. Here is an excerpt of my `pelicanconf.py`:

```python
THEME = 'themes/pure'
PATH = 'content'
# Extract a post's date from the filename:
FILENAME_METADATA = '(?P<date>\d{4}-\d{2}-\d{2})-(?P<slug>.*)'
STATIC_PATHS = ['images']
EXTRA_PATH_METADATA = {
    'images/favicon.png': {'path': 'favicon.png'},
}
# Markdown extensions:
MD_EXTENSIONS = ['codehilite(css_class=highlight,'
                 'guess_lang=False,linenums=False)',
                 'headerid',
                 'extra']
# Pagination:
DEFAULT_PAGINATION = 5
PAGINATION_PATTERNS = (
    (1, '{base_name}/', '{base_name}/index.html'),
    (2, '{base_name}/page/{number}/', '{base_name}/page/{number}/index.html'),
)
PLUGIN_PATHS = ['pelican-plugins', 'plugins']
# Pelican plugins:
PLUGINS = [# These plugins are part of the official `pelican-plugins` repo:
           'render_math',
           'summary',
           'neighbors',
           # This one is a custom plugin:
           'pdf',
           ]
ARTICLE_URL = '{slug}/'
ARTICLE_SAVE_AS = '{slug}/index.html'
PAGE_URL = '{slug}/'
PAGE_SAVE_AS = '{slug}/index.html'
```

At this point, develop and configure your site locally until it's ready to be made public. I generally use `make html` to generate the website locally, `make regenerate` to have it regenerated automatically while I work on it, and `make serve` to browse it locally at `http://localhost:8000`.

The `publishconf.py` makes a few adjustements to your `pelicanconf.py` to make your website ready to be published (mainly specifying the public URL of your website).


## Automatically generating a PDF version of my CV

One of the pages of my site contains my CV in Markdown. I wanted to have a PDF version automatically available, using [pandoc](http://pandoc.org/) and LaTeX to convert from Markdown to PDF. I created a quick-and-dirty plugin for this purpose:

```python
# This is in `plugins/pdf/__init__.py`
import os
import tempfile

from pelican import signals

# The pandoc command. The CV is saved in a static `pdfs/` subdirectory.
CMD = ('pandoc {fn} -o content/pdfs/cv.pdf '
       '-V geometry:margin=1in '
       '--template=template.tex')


def generate_pdf(p):
    with tempfile.TemporaryDirectory() as tmpdir:
        print("Generating cv.pdf")
        with open('content/pages/about.md', 'r') as f:
            contents = f.read()
        fn = os.path.join(tmpdir, 'about.md')
        contents = contents[contents.index('\n---') + 4:]
        # Add title and author in Markdown front matter.
        contents = ('---\n'
                    'title: Curriculum vitae\n'
                    'author: Cyrille Rossant\n'
                    '---\n\n' +
                    contents)
        with open(fn, 'w') as f:
            f.write(contents)
        os.system(CMD.format(fn=fn))


def register():
    # Create the PDF before generating the site.
    signals.initialized.connect(generate_pdf)
```

Now, as part of the build process, a `content/pdfs/cv.pdf` file is automatically generated. This ensures that the PDF is always in sync with that page. This PDF is not tracked by git. It will be automatically generated by Travis CI.


## Setting up Travis CI

Now we're going to set up Travis CI. We'll tell Travis to build the website at every push to the `sources` branch, and to force push the output to the `master` branch. This ensures that the website is automatically built and deployed.

Here's my `.travis.yml`:

```yaml
language: python
python:
  - "3.5"
sudo: required
services:
  - docker
branches:
  only:
  - sources
env:
  global:
    secure: "xxxxxxxxxxxx"
install:
- pip install ghp-import
- git clone https://github.com/getpelican/pelican-plugins
script:
- make publish github
```

A few things to note:

* I use Docker to build the website and the PDF but this is optional.
* If you don't use Docker, you'll have to install Pelican and other dependencies to build your website.
* I put an encrypted version of an authentication key to allow Travis to push to the `master` branch of the repo. Refer to [this page](http://blog.mathieu-leplatre.info/publish-your-pelican-blog-on-github-pages-via-travis-ci.html) to see how to generate and encrypt an authentication key.
* I use the [`ghp-import` tool](https://github.com/davisp/ghp-import) to push the generated website to the `master` branch. **Note that this tool is destructive: here it will destroy your `master` branch every time**. You will always have a single commit in `master` with the latest version of your website.
* The build process occurs in `make publish github` which is readily provided by the default `Makefile`. What this command does is:
    * Generate your website in `output/`.
    * Commit the `output/` to the `master` branch.
    * Push force that branch to GitHub. GitHub Pages takes care of the rest and updates your website automatically at `http://yourname.github.io`.


## Setting up Docker

The default `Makefile` contains the command `pelican contents/ -o output/ -s publishconf.py` to generate your website. However, since I'm using Docker, I've replaced this command by a `bash run_docker.sh`, described below.

The main reason why I'm using Docker here is that installing LaTeX takes a while, and using Docker makes the build process slightly faster on Travis CI. The image is big though (almost 1GB), mainly because of LaTeX, and I'd be happy to find a way to make it smaller. It would make the build process faster.

Using Docker also gives me a bit more control on the dependencies I need. But it certainly makes the setup more complicated. Don't use it if you don't need it.

First, install Docker locally. This is not necessarily straightforward: [follow all instructions](https://docs.docker.com/engine/installation/). Also, create a [Dockerhub](https://hub.docker.com/) account, and create a `yourname/pelican` repository. Dockerhub is like GitHub, but for Docker images.

Then, create a `Dockerfile` at the root of your repo with the following:

```
FROM python:3
MAINTAINER yourname <your@email.com>

# Update OS
RUN apt-get update
RUN sed -i 's/# \(.*multiverse$\)/\1/g' /etc/apt/sources.list
RUN apt-get -y upgrade

# Install dependencies
# I need LaTeX and pandoc to generate the CV:
RUN apt-get install make git tex-common texlive pandoc -y
RUN pip install pelican Markdown ghp-import
RUN pip install --upgrade pelican Markdown ghp-import

WORKDIR /site
# Generate the website when running the container:
CMD pelican content/ -o output/ -s publishconf.py
```

Starting from a Python 3 image, we add LaTeX, pandoc, Pelican, Markdown, and ghp-import, and we generate the website.

When you'll run a container based on this image, you'll have to mount your repository as a data volume so that the Docker container has access to it.

Build your container with `docker build -t yourname/pelican .` (note the trailing dot!). This will download the Python 3 image and build an image with your Dockerfile instructions.

Next step is to [upload the image to your Dockerhub account](https://docs.docker.com/engine/userguide/dockerrepos/) with `docker login` and `docker push yourname/pelican`. Travis CI will download it and use it to build your website.

Finally, here's a tiny bash script to pull the latest version of the image from Dockerhub and run it to generate the website:

```bash
# This is run_docker.sh
# Pull the latest Docker image
docker pull yourname/pelican
# Generate the site with pelican
docker run -t -v $(pwd):/site yourname/pelican
```

The last line of the script runs our container. The `-v $(pwd):/site` allows us to mount the current directory (typically your `~/git/yourname.github.io/` repo) to the `/site` directory, which is our container's working directory.

Phew, that's it! Now I can edit the Markdown sources, commit and push to GitHub, and the website is built automatically by Travis CI. To sum up, the build process done by Travis CI at every push to `sources` is as follows:

* Clone the current `yourname.github.io` repo
* Pull the `yourname/pelican` image from Dockerhub
* Create and run a container based on this image, with the current directory containing the sources mounted inside the container
* The container, which has Pelican, LaTeX, and pandoc installed, generates the website in `output/`, including the PDF version of one of the pages
* The output is committed to the `master` branch via `ghp-import`
* The `master` branch is pushed to the GitHub `yourname.github.io` repo thanks to the authentication token