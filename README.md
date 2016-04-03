# gostatic

Gostatic is a static site generator. What distinguishes it from most of other tools is
that it's written in Go and tracks changes, which means it should work
reasonably [fast](#speed).

Features include:

 - No run-time dependencies, just a single binary - download it and run
 - Dependency tracking and re-rendering only changed pages
 - Markdown support
 - Flexible [filter system](#processors)
 - Simple [config syntax](#configuration)
 - Support for pagination
 - Plays well with external scripts
 - HTTP server and watcher (instant rendering on changes)
 - Suitable for automation (ability to query state with `gostatic --dump`)

And all in all, it works nicely for me, so it may work for you!

## Installation

If you're Go user and want to install this from source, you know what to do (`go
get` it).

In other case, download a binary from
[releases page](https://github.com/piranha/gostatic/releases) - which also
serves as **CHANGELOG**.

If you need to automate downloading latest release, I use this script (change
`64-linux` to the type you need):

```
URL=$(curl -s https://api.github.com/repos/piranha/gostatic/releases | awk '/download_url.*64-linux/ { print $2; exit }')
curl -s $(URL) -o gostatic
```

## Quick Start

Run `gostatic -i my-site` to generate basic site in directory `my-site`. It will
have a basic `config` file, which you should edit to put relevant variables at
the top - it also contains description of how files in your `src` directory are
treated.

`src` directory obviously contains sources of your site (name of this directory
can be changed in `config`). You can follow general idea of this directory to
create new blog posts or new pages. All files, which are not mentioned in
`config`, are just copied over. Run `gostatic -fv config` to see how your `src`
is processed.

`site.tmpl` is a file that defines templates your are able to use for your
pages. You can see those templates mentioned in `config`.

And, finally, there is a `Makefile`, just for convenience. Run `make` to build
your site once or `make w` to run watcher and server, to see your site changes
in real time.

Also, you could look at [my site](https://github.com/piranha/solovyov.net) for
an example of advanced usage.

Good luck! And remember, your contributions either to gostatic or to
documentation (even if it's just this `README.md`) are always very welcome!

## Documentation index:

- [Approach](#approach)
- [Speed](#speed)
- [External Resources](#external-resources)
- [Configuration](#configuration)
  - [Constants](#constants)
- [Page Config](#page-config)
- [Processors](#processors)
- [Template API Reference](#template-api-reference)
  - [Global Functions](#global-functions)
  - [Page interface](#page-interface)
  - [Page list interface](#page-list-interface)
  - [Site interface](#site-interface)
- [Extensibility](#extensibility)

Also, see [wiki](https://github.com/piranha/gostatic/wiki) - and feel free to
add more information there!

## Approach

Each given file is processed through a pipeline of filters, which modify the
state and then rendered on disk. Single input file corresponds to a single
output file, but filters can generate virtual input files.

Each file can have dependencies, and will be rendered in case it does not exist,
or its source is newer than output, or one of this is the case for one of its
dependencies.

All read pages are sorted by date. This date is taken in their
[config](#page-config) or, in case if config is absent or dates there are equal,
by file modification time.

## Speed

On late 2008 MacBook (C2D 2.4 GHz, 8 GB RAM, 5400 rpm HDD) it takes `0.3s` to
generate a site of 250 pages. It costs `0.05s` to check there are no
modifications and `0.1s` to re-render a single changed page (along with index
and tag pages, coming to 77 pages in total).

Same timings on late 2012 MacBook Air (i7 2.0, 8 GB RAM, SSD) - `0.18s`, `0.03s`
and `0.08s`. Also note that if you're using various external post-processors
(like uglifyjs or lessc) they tend to slow down things a bit (for my specific
use case both uglifyjs and lessc add another `0.4s` when files they process
change).

## External resources

 - Jack Pearkes made a [Heroku buildpack][] for gostatic.

[Heroku buildpack]: https://github.com/pearkes/heroku-buildpack-gostatic

## Configuration

Config syntax is Makefile-inspired with some simplifications, look at the
example:

```Makefile
TEMPLATES = site.tmpl templates-folder
SOURCE = src
OUTPUT = site

# this is a comment
*.md:
    config
    ext .html
    directorify
    tags tags/*.tag
    markdown
    template page # yeah, this is a comment as well

index.md: blog/*.md
    config
    ext .html
    inner-template
    markdown
    template page

*.tag: blog/*.md
    ext .html
    directorify
    template tag
    markdown
    template page
```

Here we have constants declaration (first three lines), a comment and then three
rules. One for any markdown file, one specifically for index.md and one for
generated tags.

Note: Specific rules override generic matching rules, but logic is not exactly
very smart, and there is no real precedence defined, so if you have several
matches for a single file you could end up with any of them. Note that there is
some order: exact path match, exact name match, glob path match, glob name
match. NOTE: this may change in future.

Rules consist of path/match, list of dependencies (also paths and matches, the
ones listed after colon) and commands.

Each command consists of a name of processor and (possibly) some
arguments. Arguments are separated by spaces.

Note: if a file has no rules whatsoever, it will be copied to exactly same
location at destination as it was in source without being read into memory. So
heavy images etc shouldn't be a problem.

### Constants

There are three configuration constants:

- `SOURCE` - sources to read (relative to location of config)
- `OUTPUT` - directory for output (relative to location of config)
- `TEMPLATES` - list of files and/or directories (containing `*.tmpl` files),
which will be parsed as Go templates. Each file can contain more than one
template (see [docs](http://golang.org/pkg/text/template/#hdr-Nested_template_definitions)
on that).

You can also use arbitrary names for constants to
[access later](#site-interface) from templates - just use any other name
(`AUTHOR` could be one).

All constants can also be accessed from the config itself, using
`$(CONSTANT_NAME)` syntax, just like in `Makefile`.

## Page Config

Page config is only processed if you specify `config` processor for a page. It's
format is `name: value`, for example:

```
title: This is a page
tags: test
date: 2013-01-05
```

Parsed properties:

- `title` - page title.
- `tags` - list of tags, separated by `,`.
- `date` - page date, could be used for blog. Accepts formats from bigger to
  smaller (from `"2006-01-02 15:04:05 -07"` to `"2006-01-02"`)
- `hide` - false if not specified or is one of `f`, `false`, `False`,
  `FALSE`. True in other cases. Hides page from children and tag lists when true.

You can also define any other property you like, it's value will be treated as a
string and it's key is capitalized and put on the `.Other`
[page property](#page-interface).

## Processors

You can always check list of available processors with `gostatic --processors`.

- `config` - reads config from content. Config should be in format "name: value"
  and separated by four dashes on empty line (`----`) from content.

- `ignore` - ignore file.

- `rename <new-name>` - rename a file to `new-name`. New name can contain `*`,
  then it will be replaced with whatever `*` captured in path match. Right now
  rename touches **whole** path, so be careful (you may need to include whole
  path in rename pattern) - *this may change in future*.

- `ext <.ext>` - change file extension to a given one (which should be prefixed
  with a dot).

- `directorify` - rename a file from `whatever/name.html` to
  `whatever/name/index.html`.

- `markdown` - process content as Markdown.

- `inner-template` - process content as Go template.

- `template <name>` - pass page to a template named `<name>`.

- `tags <path-pattern>` - create a virtual page for all tags of a current
  page. This tag page has path formed by replacing `*` in `<path-pattern>` with
  a tag name and has a tag as its `.Title` (use `{{ range .Site.Pages.WithTag
  .Title }}...{{end}}` to get a list of tagged pages.

- `relativize` - change all urls archored at `/` to be relative (i.e. add
  appropriate amount of `../`) so that generated content can be deployed in a
  subfolder of a site.

- `external <command> <args...>` - call external command with content of a page
  as stdin and using stdout as a new content of a page. Has a shortcut:
  `:<command> <args...>` (`:` is replaced with `external `).

- `paginate <n> <path-pattern>` - create a virtual page for each `n` of pages
  (grouped by `path-pattern`, so you can paginate few groups of pages as a
  single one). `path-pattern` has `*` replaced by an index of this virtual page
  (1-based), and you can get a list of pages with `{{ range paginator
  .}}...{{end}}` (see [paginator](#global-functions) function). Using `paginate`
  with the same `path-pattern` on different types of pages will group them in
  same paginated list (*request*: please open an
  [issue](https://github.com/piranha/gostatic/issues/new) if you have an idea
  how to phrase this better).

## Template API Reference

Templating is provided using
[Go templates](http://golang.org/pkg/text/template/). See link for documentation
on syntax.

Each template is executed in context of a [page](#page-interface). This means it
has certain properties and methods it can output or call to generate content,
i.e. `{{ .Content }}` will output page content in place.

### Global functions

Go template system provides some convenient
[functions](http://golang.org/pkg/text/template/#hdr-Functions), and gostatic
expands on that a bit:

- `changed <name> <value>` - checks if value has changed since previous call
  with the same name. Storage used for checking is global over the whole run of
  gostatic, so choose unique names for different places.

- `cut <begin> <end> <value>` - cut partial content from `<value>`, delimited
  by regular expressions `<begin>` and `<end>`.

- `hash <value>` - return 32-bit hash of a given value.

- `version <page> <path>` - return relative URL to a page with resulting path
  `<path>` with `?v=<32-bit hash>` appended (use to override cache settings for
  static files).

- `truncate <length> <value>` - truncate string to given length (if it's
  longer).

- `strip_html <value>` - remove all HTML tags from string.

- `strip_newlines <value>` - remove all line breaks and newlines from string.

- `replace <old> <new> <value>` - replace all occurrences of `old` with `new`.

- `replacen <old> <new> <n> <value>` - same as above, but only `n` times.

- `split <separator> <value>` - split string by separator, generating an array
  (you can use `range` with result of this function).

- `contains <needle> <value>` - check if a string (`value`) contains another one
  (`needle`).

- `markdown <value>` - convert a string (`value`) from Markdown to HTML.

- `paginator <page>` - get a [paginator](#paginator-interface) object for
  current page (only works on pages created by `paginate` processor).

- `exec <cmd> [<arg1> <arg2> ....]` - exec a command with (optional) arguments.

- `excerpt <text> <maxWordCount>` - Gets an excerpt from the given text, to a
  maximum of `maxWordCount` words. When the text is shortened, it will produce
  an `[...]` string, denoting there's more. For example, `The quick brown fox`
  with `maxWordCount` of 2 will result in `The quick [...]`.

### Page interface

- `.Site` - global [site object](#site-interface).
- `.Rule` - rule object, matched by page.
- `.Pattern` - pattern, which matched this page.
- `.Deps` - list of pages, which are dependencies for this page.
- `.Next` - next page in a list of all site pages (use specific PageSlice's
  `.Next` method if you need more precise matching).
- `.Prev` - previous page in a list of all site pages (use specific PageSlice's
  `.Prev` method if you need more precise matching).

----

- `.Source` - relative path to page source.
- `.FullSource` - full path to page source.
- `.Path` - relative path to page destination.
- `.FullPath` - full path to page destination.
- `.ModTime` - page last modification time.

----

- `.Title` - page title.
- `.Tags` - list of page tags.
- `.Date` - page date, as defined in [page config](#page-config).
- `.Hide` - boolean if page is going to be absent from `{{ .Children }}` or `{{
  .WithTag }}` lists.
- `.Other` - map of all other properties (capitalized) from
  [page config](#page-config), like `{{ .Other.Author }}`.

----

- `.Raw` - page content after preprocessors (i.e. after `config` has stripped it
  part), that was originally read from the disk.
- `.Content` - page content.
- `.Url` - page url (i.e. `.Path`, but with `index.html` stripped from the end).
- `.Name` - page name (i.e. last part of `.Url`).
- `.UrlTo <other-page>` - relative url from current to some other page.
- `.Rel <url>` - relative url to given absolute (anchored at `/`) url.
- `.Is <url>` - checks if page is at passed url (or path) - use it for marking
  active elements in menu, for example.
- `.UrlMatches <pattern>` - checks if page url matches regular expression
  `<pattern>`.

### Paginator interface

- `.Number` - number of paginator page, first is 1
- `.PathPattern` - whatever was passed as `path-pattern` to `paginate`
  (processor)[#processors]
- `.Page` - paginator's own [page](#page-interface)
- `.Pages` - [list of pages](#page-list-interface)
- `.Prev` - previous paginator object (if current is first, then `nil`)
- `.Next` - next paginator object (if current is last, then `nil`)

### Page list interface

- `.Get <n>` - [page](#page-interface) number `<n>`.
- `.First` - first page.
- `.Last` - last page.
- `.Len` - length of page list.
- `.Prev <page>` - return page with earlier date than given. Returns nil if no
  earlier pages exist or page is not in page list.
- `.Next <page>` - return page with later date than given. Returns nil if no
  later pages exist or page is not in page list.
- `.Slice <from> <to>` - return pages from `from` to `to` (i.e. from 0 to 10).

----

- `.Children <prefix>` - list of pages, nested under `<prefix>`.
- `.WithTag <tag-name>` - list of pages, tagged with `<tag-name>`.

----

- `.BySource <path>` - finds a page with source path `<path>`.
- `.ByPath <path>` - finds a page with resulting path `<path>`.
- `.GlobSource <pattern>` - list of pages, [matching](http://golang.org/pkg/path/#Match) source path `<pattern>`.

### Site interface

- `.Pages` - [list of all pages](#page-list-interface).
- `.Source` - path to site source.
- `.Output` - path to site destination.
- `.Templates` - list of template files used for the site.
- `.Other` - any other properties (capitalized) defined in site config.

## Extensibility

Obviously, the easiest way to extend gostatic's functionality is to use
`external` [processor](#processors). It makes you able to process files in the
way you want, but is more or less limited to that. There is no API right now to
create pages on the fly (like `tags` processor does) using this method, for
example.

But `gostatic` itself is a
[library](https://github.com/piranha/gostatic/tree/master/lib), and you can
write your own static site generator using this library. See
[gostatic.go](https://github.com/piranha/gostatic/blob/master/gostatic.go) for
an example of one.
