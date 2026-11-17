# curriculum - lua filters for vitae templates

Core Lua filters for building CVs with Quarto, providing the shared machinery
that vitae-compatible templates are built on. A language-agnostic successor to
the R package of R Markdown templates [vitae](https://github.com/mitchelloharawild/vitae/).

> [!IMPORTANT]
> **Writing a CV? You are in the wrong place.**
> 
> This repository is for people
> *building* CV templates. To use vitae, learn more at
> **<https://quarto-vitae.github.io/>**.

## What this provides

CV entry data and contact details, handled the same way across every vitae template. An author
writes a listing as a table and names a style:

```markdown
::: {.entries style="detailed"}
| what             | with                | when      |
|------------------|---------------------|-----------|
| Ph.D. in Physics | University of Paris | 1891-1895 |
:::
```

More often the data is generated, so the same options are given as cell options
in R, Python or Julia. The cell prints a table and `curriculum` picks it up
wherever it appears:

````markdown
```{r}
#| echo: false
#| vitae:
#|   style: detailed
#|   fields:
#|     what: degree
#|     with: institution
#|     when: years
education |> filter(completed)
```
````

This is the common case, and the reason for it: filtering and sorting stay in
the language the author already uses.

### How fields reach your listing files

Each row of the table becomes one entry, and each column becomes a field on it.
Columns keep their own heading by default, so `fields` is only a list of
corrections, written `new: old` like `dplyr::rename()`:

```
data column        entry field        your handler
───────────        ───────────        ────────────
degree        ──►  what          ──►  $entries.what$
institution   ──►  with          ──►  $entries.with$
years         ──►  when          ──►  $entries.when$
```

Your handler is applied once to the whole listing, looping over the entries:

```tex
\begin{cventries}
$for(entries)$
  \cventry{$entries.what$}{$entries.with$}{$entries.where$}{$entries.when$}{}
$endfor$
\end{cventries}
```

Two things follow. Values arrive already rendered and escaped for your
target format, so `**bold**` is `\textbf{bold}` and `100%` is `100\%`, and you
must not escape them again. And a column the author did not supply is simply
absent, which pandoc renders as empty, so handlers need no guards for missing
fields.

This simplifies the process of creating new vitae-compatible CV templates. All
that is needed is one small pandoc template for each listing style. 
Generic fallbacks for LaTeX, Typst, HTML and markdown mean a style you have not
implemented still renders (e.g. if a user is swapping templates that define
custom styles that your template does not).

### Bibliographies

`style: bibliography` declares that a listing's data is *citation data*
rather than a table of records, rendered by citeproc. It accepts `.bib`,
CSL-YAML, or CSL-JSON, from a whole file or inline:

| source                  | `.bib` / bibtex | CSL-YAML      | CSL-JSON      |
|--------------------------|-----------------|---------------|---------------|
| `file=` attribute        | ✓               | ✓             | ✓             |
| fenced block in the div  | ✓ ` ```bibtex ` | ✓ ` ```yaml ` | ✓ ` ```json ` |
| printed cell output      | ✓               | ✓             | ✓             |

The common case is a whole file, formatted by the document CSL:

```markdown
::: {.entries style="bibliography" file="publications.bib"}
:::
```

The programmatic path prints citation data from a cell — filter and sort in
your own language, then emit CSL back out. The format is detected
automatically (override with `format: bibtex|csljson|cslyaml` if needed), and
the cell must *not* set `output: asis`:

````markdown
```{r}
#| vitae:
#|   style: bibliography
refs <- yaml::read_yaml("publications.yaml")$references
kept <- Filter(\(x) x$type == "article-journal", refs)
cat(yaml::as.yaml(kept))
```
````

No `csl:` is set by default, so entries render in whatever order citeproc's
own default style gives them unless you set one yourself — see below.

#### Choosing a CSL style for ordering

Ordering is entirely the `<sort>` key inside a style's `<bibliography>`
element — a CV just wants a different one than citeproc's alphabetical
default. Reverse-chronological sorts on `issued` descending:

```xml
<bibliography>
  <sort>
    <key variable="issued" sort="descending"/>
  </sort>
  ...
</bibliography>
```

Removing the `<sort>` element entirely makes citeproc fall back to input
order — useful if you already filtered and sorted the entries upstream, in
R/Python/Julia, and want the bibliography to display exactly that order. No
off-the-shelf style does this in a form suited to a CV; see
[`todo-csl.md`](todo-csl.md) for what was checked and what building one would
take.

Save the result as a `.csl` file and set `csl: mystyle.csl` in the front
matter (or in `_quarto.yml` to apply it project-wide) — or adjust one online
first in the [visual CSL editor](https://editor.citationstyles.org/visualEditor/).
[`apa-cv`](https://www.zotero.org/styles/apa-cv) is a ready-made
reverse-chronological example, usable straight from its URL:

```yaml
csl: https://www.zotero.org/styles/apa-cv
```

For template authors, bibliography entries route through the same template
layer as everything else: supply `entries/bibliography.tex` / `.typ` / `.html`
and each entry arrives with three fields — `entry` (the reference, rendered by
citeproc and written for your target format), `id` (citation key) and `type`
(CSL type, e.g. `article-journal`). No CSL knowledge enters your handler. A
format that supplies nothing gets citeproc's own output passed through
unchanged — the fallback for this style is passthrough, never the generic
layout. One LaTeX caveat: under a numbered CSL style, `entry` contains
`\CSLLeftMargin`/`\CSLRightInline`. Pandoc defines those macros only when its
own csl divs remain in the document (the passthrough case), so a format routing
bibliographies through its own handler should define them itself or prefer a
non-numbered CSL.

### Contact details

The header of a CV (address, phone, email, profiles) is front matter rather than
a listing. Every vitae template reads it from the same `social` key, so an author
can switch templates without touching it:

```yaml
social:
  - address: "Paris, France"
  - email: marie.curie@example.org
  - website: example.org
  - github: mcurie
  - mastodon: "@mcurie@fosstodon.org"
  - stackoverflow: 123456/marie-curie
  - linkedin: {handle: mcurie, label: "Marie Curie"}
  - custom:
      - icon: flask
        text: "[Curie Laboratory](https://curie.fr)"
      - icon: clock
        text: "Available Mon–Thu"
```

Each item is `service: handle`, written the way that service writes it, so a
Mastodon handle is `@user@instance`, not two separate keys. A full profile URL
works in place of any handle. A map lets the author override any of `handle`,
`url`, `label`, `icon` or `title` for that item. Entries under `custom` are
anything else, given as an icon and markdown `text`, with an optional `url`.

`social` is a list because pandoc sorts YAML map keys, so only a list
keeps the order the author wrote. A plain map (`social: {github: mcurie}`) is
accepted, and is shown in the order of the table below.

| service | handle | default link | default icon |
|---------|--------|--------------|--------------|
| `address` | free text | none | `location-dot` |
| `phone` | as written | `tel:` | `phone` |
| `email` | address | `mailto:` | `envelope` |
| `website` | URL, scheme optional | the URL | `globe` |
| `whatsapp` | phone number | `wa.me/` | `whatsapp` |
| `telegram` | username | `t.me/` | `telegram` |
| `skype` | username | `skype:…?chat` | `skype` |
| `orcid` | iD | `orcid.org/` | `orcid` |
| `googlescholar` | profile id (labelled "Google Scholar") | `scholar.google.com/citations?user=` | `google-scholar` |
| `researchgate` | profile name | `researchgate.net/profile/` | `researchgate` |
| `github` | username | `github.com/` | `github` |
| `gitlab` | username | `gitlab.com/` | `gitlab` |
| `bitbucket` | username | `bitbucket.org/` | `bitbucket` |
| `stackoverflow` | `id` or `id/name` | `stackoverflow.com/users/` | `stack-overflow` |
| `kaggle` | username | `kaggle.com/` | `kaggle` |
| `hackerrank` | username | `hackerrank.com/profile/` | `hackerrank` |
| `linkedin` | vanity name | `linkedin.com/in/` | `linkedin` |
| `mastodon` | `@user@instance` | `https://instance/@user` | `mastodon` |
| `bluesky` | handle | `bsky.app/profile/` | `bluesky` |
| `x` | username | `x.com/` | `x-twitter` |
| `twitter` | username | `twitter.com/` | `twitter` |
| `xing` | profile name | `xing.com/profile/` | `xing` |
| `reddit` | username | `reddit.com/user/` | `reddit` |
| `medium` | username | `medium.com/@` | `medium` |

An unknown service name is a warning that suggests the nearest known name,
rather than an entry that silently goes missing.

#### What a template receives

The filter replaces `social` with a list in which every item has the same
shape, whatever the author wrote:

| field | example | |
|-------|---------|-|
| `service` | `mastodon`, `custom` | |
| `is.<service>` | `is.mastodon` | true for this item's service, for `$if$` |
| `title` | `Mastodon` | the service's name, for alt text; absent on custom entries without one |
| `icon` | `mastodon` | a [Font Awesome 6](https://fontawesome.com/search?ic=free) name |
| `url` | `https://fosstodon.org/@mcurie` | absent when there is nothing to link (`address`, custom without `url`) |
| `label` | `@mcurie@fosstodon.org` | the display text |
| `handle` | `@mcurie@fosstodon.org` | the handle, recovered from a URL if one was given |
| `given.<field>` | `given.icon` | true where the author set `url`, `label`, `icon` or `title` themselves |
| parts | `user`, `instance` (mastodon); `id`, `slug` (stackoverflow); `id` (googlescholar) | only where the service has them |

As with entry fields, values arrive rendered for the target format. `url` is
escaped for a link target (`\href{}` in LaTeX, a string in Typst), and
everything else for running text, so build links from `url`, never from
`handle`. A custom `label` may already contain a link of its own.

The whole contact line is then one loop:

```tex
$for(social)$
$if(social.url)$\href{$social.url$}{\faIcon{$social.icon$}~$social.label$}$else$\faIcon{$social.icon$}~$social.label$$endif$$sep$ | 
$endfor$
```

The icons, links and labels are defaults. A template overrides them with
pandoc's own conditionals, which is also how it takes an item out of the
contact line to place it elsewhere:

```tex
$for(social)$
$if(social.is.address)$
\address{$social.label$}
$else$
\cvsocial{$if(social.given.icon)$\faIcon{$social.icon$}$elseif(social.is.github)$\faSquareGithub$else$\faIcon{$social.icon$}$endif$~$social.label$}
$endif$
$endfor$
```

Check `given.<field>` before substituting a default of your own, as above, so
that an icon or label the author chose still wins over the template's.

Pandoc's `$sep$` cannot know an item was skipped, so a template that moves items
out of the loop should insert its separators itself, as LaTeX's own list
macros do.

## Using these filters in a template

A CV template is an ordinary Quarto format extension that embeds `curriculum`
and adds a directory of listing handlers:

```
_extensions/yourcv/
├── _extension.yml
├── _extensions/
│   └── quarto-vitae/
│       └── vitae-entries/     <- embedded copy of curriculum
└── entries/
    ├── detailed.tex
    ├── brief.tex
    └── skills.tex
```

### Embedding curriculum

Quarto cannot resolve one extension from another by name alone, so a template
that is distributed separately must carry its own copy. This is what
`quarto add --embed` is for. From the root of your template repository:

```bash
quarto add quarto-vitae/curriculum --embed yourcv
```

The first argument is this repository, the `--embed` argument is the directory
name of your own extension under `_extensions/`. The copy lands under your
extension's own `_extensions/`, and `filters: [vitae-entries]` then resolves
from anywhere in that tree.

> [!NOTE]
> Add the dependency *before* you write `filters: [vitae-entries]`. Quarto
> validates the target extension while installing into it, so a reference to a
> filter that is not there yet makes the install fail with the same
> "Failed to resolve referenced filter" error it is meant to cure. This applies
> only to the first `add`; updates are unaffected.

Commit the embedded copy. It is part of your extension as distributed.

### Staying up to date

To pick up a later release of `curriculum`, re-embed with the same pair of
arguments:

```bash
quarto update quarto-vitae/curriculum --embed yourcv
```

This is a clean replace, not an overlay: files dropped upstream are removed
rather than left behind. Commit the result and cut a release of your template.

Updates reach CV authors in two hops, and only the second is theirs:

```
curriculum release
  -> template re-embeds and releases     quarto update ... --embed yourcv
    -> author updates the template       quarto update quarto-vitae/yourcv
```

Updating the template refreshes the copy of `curriculum` inside it, so authors
need no separate step and have no way to update the filter on their own. Until
1.0, nothing warns about a stale embedded copy, which makes re-embedding
promptly your job rather than theirs.

### Listing handlers

Name each handler after the style it renders, with the extension of the target
format: `.tex` for LaTeX, `.typ` for Typst, `.html`, or `.md` for anything else.
Ship one per format you support.

Adding a style is one file plus one line of YAML. The filter needs no change.

### Style conventions

Style names describe a listing shape, not a section of a CV. The same shape 
serves education, employment and awards alike, and which sections a CV has is
the author's business, not the template's. So `detailed` and `brief` are style
names, while `education` and `employment` are not.

Every vitae-compatible template should implement at least these two:

`detailed`
: A multi-line entry: a heading and subheading, a date, a place, and a bulleted
  elaboration built from `why`. This is the workhorse, used for education and
  employment. It is also the style assumed when an author gives none, so a
  template without it will fail on the simplest possible document.

`brief`
: A single line per entry, typically just `what` and `when`, with no
  elaboration. Used for awards, memberships and service.

Providing both is what makes templates interchangeable. An author switches
design by editing one line of YAML, and that only works if a style name means
the same shape everywhere. A template that implements only its own inventions
cannot be swapped into an existing CV.

Beyond those two, `skills` (a category paired with its members) is common enough
to be worth matching if your design suits it. Invent a new name only when the
shape is genuinely new, such as `publications` or `timeline`, and declare
everything you ship in `vitae.styles` so authors get a useful message instead of
an unstyled fallback.

### `_extension.yml`

Alongside whatever your format already needs (a document class, a pandoc
template, fonts), each contributed format sets three vitae keys:

```yaml
contributes:
  formats:
    <format>:
      filters:
        - vitae-entries
      df-print: kable
      vitae:
        styles: [detailed, brief, ...]
```

Repeat these for every format you contribute, so that a template offering both
`pdf` and `typst` declares them under each.

`filters: [vitae-entries]`
: Runs the filter. Without it, `.entries` divs pass through untouched.

`vitae.styles`
: The listing styles your format implements. This is not how styles are
  discovered, since handlers are found by filename. It is how the filter tells a
  missing handler from an author's typo, so that each gets a useful message
  rather than a silent fallback.

`df-print: kable`
: Optional but recommended. Lets a knitr cell print a data frame and still
  produce a table, so authors need no `knitr::kable()` call. It is a format
  option, ignored by Quarto if set per cell, so opting in is the template's job.

> [!WARNING]
> Do not put `entries` in `format-resources`. Resources are copied next to the
> rendered document *after* pandoc runs, so a handler listed there resolves only
> on the second render, appearing to work locally while failing in CI. The
> filter reads `_extensions/**/entries/` directly, so leave the directory where
> it is.

## Full guide

**<https://quarto-vitae.github.io/>** documents template development in full:
every `_extension.yml` option, how handlers are resolved and overridden, the
template context and its pandoc-template gotchas, and conventions for field and
style names.

[quarto-vitae/awesome-cv](https://github.com/quarto-vitae/awesome-cv) is a
working template to read alongside it.

## Licence

MIT. See [LICENSE](LICENSE). Templates embedding these filters may be licensed
however their own design requires.
