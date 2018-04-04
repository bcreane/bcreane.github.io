---
layout: post
title:  "Google App Engine - suitable for static sites?"
date:   2018-04-03 20:41:11 -0700
categories: google
---

# TL;DR
Google App Engine is very easy to provision - automatic tls certs, friendly
domain name, caching, scaling and robust debugging are among the useful features.

However using GAE to serve a static Jekyll site (admittedly not it's most
important use-case) has issues ranging from annoying to serious. 

Before deploying a static site on GAE, make sure the issues outlined below
are not blockers for you.

# Deploying your Jekyll site on GAE

[Jekyll](https://jekyllrb.com) creates a `_site/` directory with all of your site's markdown rendered
to `.html`. Google App Engine takes a [app.yaml](https://cloud.google.com/appengine/docs/standard/python/config/appref)
description of your site and publishes the interesting relevant files.

# GAE issue: differentiating files and directories

One nice feature of [Jekyll](https://jekyllrb.com)
is [Permalink style](https://jekyllrb.com/docs/permalinks/#builtinpermalinkstyles)
which let's you link to your pages without annoying suffixes such as `.html`.
Unfortunately GAE doesn't handle this well. 
  
Most webservers can handle all three of the following scenarios:

| Type       | URL         | Proper Webserver Serves  | GAE Serves            | Makes sense? |
| ---------- | ----------- | ------------------------ | --------------------- | ------------ |
| directory  | \_site/fun/ | \_site/fun/index.html    | \_site/fun/index.html | yep          |
| directory  | \_site/fun  | \_site/fun/index.html    | \_site/fun            | ???          |
| file       | \_site/fun  | \_site/fun.html          | \_site/fun            | yep          |

There is a workaround ... you have to tell GAE about every directory and every file in your site!

```
#!/bin/bash

# Script to generate the site yaml for GAE to serve your site
# Get all the unique file suffixes in the _site; e.g. ".txt", ".png".
# Output in form "js|html|yml", etc.
suffixes=`find _site -type f -iname \*.* -print | sed 's/.*\.//' | sort | uniq | paste -sd "|" -`

# Create a static file handler based on all the suffixes
printf -- "- url: /(.*\\.(%s))$\n" $suffixes
printf "  static_files: _site/\\\1\n"
printf "  upload: _site/(.*\\.(%s))$\n" $suffixes

```

Now let's handle the case where someone forgets to append a `/` to a directory URL,
for example: `https://_site/fun` when they mean to say `https://_site/fun/`

```
#!/bin/bash

# Get all the directories in the _site; e.g. "fun"
# Output in form "/dir1|/dir2", etc.
directories=`find _site -type d -print | sed 's/_site\///g' | sort | uniq | grep -v _site | paste -sd "|" -`

# Create a handler for URLs with a directory that do NOT have a terminal /.
printf "\n"
printf "# Handle any directory URLs that are missing a terminal /\n"
printf "#\n"
printf -- "- url: /(%s)$\n" $directories
printf "  static_files: _site/\\\1/index.html\n"
printf "  upload: _site/(%s)/index.html\n" $directories
```

Okay, this is awkward and doesn't scale, but fortunately we have a pretty small site
with fewer than 100 directories and 500 files. The generated `app.yaml` is big, but
not impossibly big.

# You don't know the mime type for scripts or yaml? Really?

Most webservers give you a robust pre-populated list of mime types. There
are only about a dozen file suffixes in our site, e.g. `yaml`, `txt`, `html`, `jpg`.
Nothing exotic.

Turns out GAE's webserver doesn't know what to do with `.bash`, `.sh`, or `.yaml`.
Again, not a big deal, but our script is getting bigger:

```
# Serve up static files based on suffix. See:
#   https://www.iana.org/assignments/media-types/media-types.xhtml
#
# Specify mime-type for yaml files since GAE doesn't handle this correctly.
#
- url: /(.*\.(yaml|yml))$
  static_files: _site/\1
  mime_type: text/x-yaml
  upload: _site/(.*\.(yaml|yml))$

# Specify mime-type for sh files since GAE doesn't handle this correctly.
#
- url: /(.*\.(sh|bash))$
  static_files: _site/\1
  mime_type: text/x-shellscript
  upload: _site/(.*\.(sh|bash))$

# For all remaining files, let GAE infer mime-type
```

# The last straw - GAE sends compressed files (when the client doesn't expect it).

This one drove me crazy for a little while. Trying to download a yaml file:
`curl -O my_kubernetes_manifest.yaml` returned binary junk in the file!

It turns out GAE _sometimes_ sends back gzipped data, even if the request header of
the client doesn't say it can accept it. But only on `https`, not `http`. And only
sometimes.

There's a one-year old bug [Google frontend serves gzipped content even if the client doesn't ask for it](https://issuetracker.google.com/issues/37938470) which captures this issue perfectly.

There are other interesting, and unresolved bugs around this area too. Try the handy [search GAE issues](https://issuetracker.google.com/issues?q=app%20engine%20curl%20https).

The workaround is to force curl to expect a gzipped response:  `curl --compressed -O my_kubernetes_manifest.yaml`

# Conclusion

GAE is a subpar approach for serving static content for the reasons outlined above (and others).
More worryingly, the slow response to outstanding bugs implies Google's attention is elsewhere.

Justin Krause did a nice job of [reviewing GAE for python dynamic sites](https://hackernoon.com/going-gae-our-experience-with-google-app-engine-deaf2b7171c1). GAE covered their needs nicely. As long as you're using GAE for exactly what
it was designed for, and you don't mind working around bugs, GAE may work for you.

## Serving a Jekyll site with Google App Engine

Here's the bash script for generating the GAE site yaml in it's entirety.

```
#!/bin/sh

# Script that inspects the jekyll-generated _site and emits
# a Google App Engine service specification file on stdout.

if [ ! -d _site ]; then
	echo _site/ doesn\'t exist yet, make sure you run \"jekyll build\".
	exit 1
fi

cat <<EOF
# A Google Application Engine "service" definition. The reference
# to python is a red-herring - we're just statically serving the
# contents of the _site subdirectory. GAE isn't setup to do this
# easily - we need to tell it how to handle each type of file, as
# well as infer "index.html" when we just refer to a site directory.
#
runtime: python27
api_version: 1
threadsafe: yes

handlers:
# Serve up static files based on suffix. See:
#   https://www.iana.org/assignments/media-types/media-types.xhtml
#
# Specify mime-type for yaml files since GAE doesn't handle this correctly.
#
- url: /(.*\.(yaml|yml))$
  static_files: _site/\1
  mime_type: text/x-yaml
  upload: _site/(.*\.(yaml|yml))$

# Specify mime-type for sh files since GAE doesn't handle this correctly.
#
- url: /(.*\.(sh|bash))$
  static_files: _site/\1
  mime_type: text/x-shellscript
  upload: _site/(.*\.(sh|bash))$

# For all remaining files, let GAE infer mime-type
#
EOF

# Get all the unique file suffixes in the _site; e.g. ".txt", ".png".
# Output in form "js|html|yml", etc.
suffixes=`find _site -type f -iname \*.* -print | sed 's/.*\.//' | sort | uniq | paste -sd "|" -`

# Create a static file handler based on all the suffixes
printf -- "- url: /(.*\\.(%s))$\n" $suffixes
printf "  static_files: _site/\\\1\n"
printf "  upload: _site/(.*\\.(%s))$\n" $suffixes

# Enumerate all the directories in the _site; e.g. "fun/". Output in form "/dir1|/dir2", etc.
directories=`find _site -type d -print | sed 's/_site\///g' | sort | uniq | grep -v _site | paste -sd "|" -`

# Create a handler for URLs with a directory that do NOT have a
# terminal /. This is a fail-safe in case someone misconstructs
# the URL without a terminal /.
printf "\n"
printf "# Handle any directory URLs that are missing a terminal /\n"
printf "#\n"
printf -- "- url: /(%s)$\n" $directories
printf "  static_files: _site/\\\1/index.html\n"
printf "  upload: _site/(%s)/index.html\n" $directories

cat <<EOF

# Default directory/html append rules
#
- url: /
  static_files: _site/index.html
  upload: _site/index.html

# For directories indicated by a terminal /, append "/index.html"
- url: /(.+)/
  static_files: _site/\1/index.html
  upload: _site/(.+)/index.html
  expiration: "15m"

# For md files, append ".html"
- url: /(.+[a-z0-9])
  static_files: _site/\1.html
  upload: _site/(.+[a-z]).html
  expiration: "15m"

- url: /(.+)
  static_files: _site/\1/index.html
  upload: _site/(.+)/index.html
  expiration: "15m"

- url: /(.*)
  static_files: _site/\1
  upload: _site/(.*)

libraries:
- name: webapp2
  version: "2.5.2"
EOF

```
