# GigaSpaces XAP Documentation

This repository contains the markup files, html templates and javascript sources for the new [GigaSpaces XAP documentation portal](http://docs.gigaspaces.com).
It's based on [Jekyll](http://jekyllrb.com/), a Ruby-based static site generator, and uses [Markdown]() as a markup language. It was originally ported from Atlassian Confluence, and therefore multiple useful Jekyll [plugins](http://docs.gigaspaces.com/howto/plugin.html) were defined. The plugins mimic the behavior of original Confluene macros to allow for easy migration from Confulence and provide content editor with useful markup extensions that are relevant to online documentation portals.

## Help Us Improve! 

It's important for us to encourage your feedback and contribution. Contributing to this website is straightforward. Simply fork this repository, make your changes, test them with your local Jekyll installation, and submit a pull request. We promise to review and comment on every pull request, and merge it if it makes sense.

## Installing, Editing and Testing Locally

To run and test the website locally, you should perform the following steps:

* Make sure Ruby and RubyGems are installed.

* Install the latest version of Jekyll (you can find detailed directions [here](http://jekyllrb.com/docs/installation/)).

  * If you're using Windows, you may want to use the (unofficial) [portable jekyll for Windows](http://www.madhur.co.in/blog/2013/07/20/buildportablejekyll.html).

* Install the [Kramdown markdown converter](http://kramdown.gettalong.org/installation.html).

* If you intend to edit the docs, create a fork of this repo and clone it to your machine.

        git clone https://github.com/&lt;your github username&gt;/gigaspaces-wiki-jekyll.git

  If you just want to run it locally, you can clone this repository:

        git clone https://github.com/Gigaspaces/gigaspaces-wiki-jekyll.git

* cd to the clone directory: 

        cd gigaspaces-wiki-jekyll

* If you're using Windows, run `setup-win.bat` with administrative permissions (requires Windows Vista or later).
This script is a workaround for a known issue in git which ignores symbolic links on windows. 

* Make the required changes to the docs (if you need to).

* Run Jekyll in sever mode, and wait for the site generation to complete: 

        jekyll serve

  You should see the following output if everything was ok:

        Configuration file: /Users/uri1803/dev/gigaspaces-wiki-jekyll/_config.yml
                    Source: /Users/uri1803/dev/gigaspaces-wiki-jekyll
               Destination: /Users/uri1803/dev/gigaspaces-wiki-jekyll/_site
              Generating... done.
            Server address: http://0.0.0.0:4000
          Server running... press ctrl-c to stop.

* Point your browser to [http://localhost:4000](http://localhost:4000). You should see the documentation portal home page, and verify that you're ok with your changes. 

* If you've forked the repository and updated documentation pages, you can submit a pull request. We will review, comment and merge the pull request after approving it. 

## Continuous Deployment 

This website is hosted on AWS S3. Every push to this reposiroty triggers a build process (currently we use the excellent [Circle CI](http://circleci.com) conitnuous integration service), at the end which the generated website is pushed to S3 using the [s3_website](https://github.com/laurilehmijoki/s3_website) library. The Circle CI configuration is located at in the file [`circle.yml`](circle.yml). 

## Authoring Guidelines

You can find these guidelines [here](http://docs.gigaspaces.com/howto)














