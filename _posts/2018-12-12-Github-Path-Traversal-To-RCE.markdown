---
layout: post
title:  "Path Traversal in GitHub pages, and more for GitHub Enterprise"
date:   2019-01-01 12:34:11 +0800
categories: jekyll update
---

# Brief
I started to write blogs (again) from early 2018. I had a blog constructed with
[obtvse](https://github.com/natew/obtvse), and added support to the
mark-up syntax of [Org mode](https://orgmode.org/) (yes I'm an Emacs user) to
make myself happy. But this time I was gonna try something new,
like [Jekyll](https://jekyllrb.com) hosted on GitHub pages, which is easy to
setup and control. So I spent like 20 minutes setting up a minimal blog powered
by Jekyll, then after several months when I finally had some
time to look into this blog and to do some customization, the story begins.

# Jekyll and symbolic links
After going through source code of Jekyll briefly I got some basic knowledge
about Jekyll. Each build of a Jekyll based website consists of steps called
`reset read generate render cleanup write`. Basically it reads all necessary
files required by the build, like layouts and static resources, then renders
the contents with these previously read resources. Here comes my question,
Is that possible to make Jekyll unconsciously load some external file (like
/etc/passwd) as a layout and use it to render some other whatever contents,
so that I could smuggle the content of sensitive files to the built website.

So I created a link `_layout/passwd.html -> /etc/passwd` and a markdown file
`passwd.md` states to be rendered by layout passwd.

``` text
---
layout: passwd
---
```

at the same time, according to `lib/jekyll/entry_filter.rb`

``` ruby
    def filter(entries)
      entries.reject do |e|
        special?(e) || backup?(e) || excluded?(e) || symlink?(e) unless included?(e)
      end
    end
```

I explicitly included `passwd.html` in `_config.yml`

After the preparation, I executed `jekyll build` on my local machine and
successfully got a rendered page `_site/passwd.html` having the same content as
`/etc/passwd`.

However when I tried this on github.com, it failed to build with the following
error message:

```
"The symbolic link _layouts/passwd.html targets a file which does not exist within your site's repository."
```

oops, sounds like GitHub doesn't like me to have a symbolic link in this
repository. Can I bypass it?

# Jekyll-remote-theme
According to [this announcement](https://blog.github.com/2017-11-29-use-any-theme-with-github-pages/)
GitHub pages supports jekyll-remote-theme plugin which allows a Jekyll site to
use other GitHub.com hosted Jekyll theme directly, without storing a copy
of the layout files in my own repository. This is an awesome feature for people
who likes customization and, of course, me.

So I created another repository with the same link part `_layout/passwd.html -> /etc/passwd`.
The repository was at [nyangawa/theme](https://github.com/nyangawa/theme). Then
I modified my own `_config.yml` to add `remote_theme: nyangawa/theme`. Visiting
`https://REDACTED.github.io/passwd.html` showed me the content of GitHub's `/etc/passwd`.
(This website was deleted after I reported the issue. I didn't reserve an screenshot neither :p)

However, GitHub pages service builds every site in an isolated environment
(docker container). There is actually very few information to grab from it.
But this is not an end yet.

# GitHub Enterprise
[GitHub Enterprise](https://enterprise.github.com/) is a self-hosted version of GitHub.
I downloaded an trial license for security assessment, so I decided to replicate
this bug in GitHub Enterprise to see whether there is any difference between GitHub.com
and GitHub Enterprise.

About deciphering the source code of GitHub Enterprise. @orangetw's [article](https://blog.orange.tw/2017/01/bug-bounty-github-enterprise-sql-injection.html)
explained this perfectly. I'm not going to explain that again, except that `ruby_concealer.so`
is part of the `ruby` executable in the VM since a recent version. So there is
no longer `require "ruby_concealer.so"` in the source code.

After pushing the repository to my GitHub Enterprise, I found that things are
different. There's no isolation here.

![fstab](/assets/2018-12-12-GitHub-Path-Traversal-To-RCE_1.png)

source code in `/data/pages/current/script/page-build` proves my guess.

``` ruby
# Run non-git commands as least-privledged user in dotcom
if [ -z "$SKIP_COMMAND_PREFIX" ] && [ "$PAGES_ENV" != "test" ] && [ "$PAGES_ENV" != "cucumber_test" ] && [ "$PAGES_ENV" != "development" ] && [ "$PAGES_ENV" != "enterprise" ]; then
  COMMAND_PREFIX="sudo -u jekyll"
fi
```

My memory brings to me [This blog](https://www.exablue.de/blog/2017-03-15-github-enterprise-remote-code-execution.html)
by iblue. And yes I can read the session secret now.

![secrets](/assets/2018-12-12-GitHub-Path-Traversal-To-RCE_2.png)

After all, send the payload and check.

![shell](/assets/2018-12-12-GitHub-Path-Traversal-To-RCE_3.png)

# Another hit on Jekyll (Updated on 2018-10-24)

Days after the first issue, I went back to the code base of Jekyll. A strange
implementation of file path checking came into my sights.

``` ruby
# lib/jekyll/theme.rb

    def includes_path
      @includes_path ||= path_for "_includes"
    end

    def layouts_path
      @layouts_path ||= path_for "_layouts"
    end

    def sass_path
      @sass_path ||= path_for "_sass"
    end

    def assets_path
      @assets_path ||= path_for "assets"
    end
# ...

    def path_for(folder)
      path = realpath_for(folder)
      path if path && File.directory?(path)
    end

    def realpath_for(folder)
      File.realpath(Jekyll.sanitized_path(root, folder.to_s))
    rescue Errno::ENOENT, Errno::EACCES, Errno::ELOOP
      Jekyll.logger.warn "Invalid theme folder:", folder
      nil
    end
```

These functions are used to get the real path of the resource directories
(includes, layouts, etc.) in the file system. But obviously, it's not a good
idea to leave symlinks alone (again?).

Imaging if we have a link `_includes -> /etc`, what happens here?

1. `realpath_for` returns `/etc`
1. `/etc` passes the check in `path_for` and returned
1. Jekyll will use `/etc` as the source directory of layouts

We successfully escaped from Jekyll's site source again.

Being able to read any file in any directory as user `git` can do a lot of
things. Including fetching session secret from some hidden places. :P

# Some thoughts
Creating applications can be enjoyable, especially with languages like Ruby
which is extremely flexible and provides many different ways to implement a
feature. But what we should not forget about is that applications run on
operating system. If an application wants to or has to interact with the
underlay OS or file system, the developer needs to not only think the
implementation from the perspective of the application, but also the perspective
of the operating system, and the possibilities beneath the surface.

I'll share some experiences about secure coding from the perspective of
a system engineer. To conclude some common pitfalls where a newbie or
even experienced developer could encounter.

# Timeline
GitHub has a very responsive and efficient bug bounty program.

1. Aug 16th: Report arbitrary file read to [GitHub on Hackerone](https://hackerone.com/github)
1. Aug 17th: Got validation
1. Aug 17th: Report the supplementary RCE part.
1. Aug 21st: Issue fixed
1. Aug 21st: $10000 bounty from GitHub, a coupon of GitHub private repositories for
life and an invitation to [@githubbounty](https://github.com/githubbounty)
1. Aug 23rd: $2000 extra from GitHub for verifying a not properly packaged release.
1. Sep 10th: Reported another directory traversal issue in GitHub pages and got
a quick first response within a several hours
1. Sep 29th: A fixed version of GitHub enterprise was released
1. Oct 4th: $10000 bounty from GitHub for the second bug
