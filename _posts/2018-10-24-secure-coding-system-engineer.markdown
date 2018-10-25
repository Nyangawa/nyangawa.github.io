---
layout: post
title:  "Secure coding with Ruby from the perspective of a system engineer (1)"
date:   2018-10-24 15:55:11 +0800
categories: jekyll update
---

# Why I'm writing this

Since approximately half an year ago, I began to do some bug hunting stuff.
During the past several months, I learned a lot and found some vulnerabilities
in some popular software.

Due to my own background (several years of experience for being a system
engineer), I payed more attention on problems caused by the OS-related issues.
Like path sanitizing, improperly handled symbolic links and unintended system
command invocations.

So I'm going to enumerate some common pitfalls which are often neglected by
developers and hard to discover by code analysis tools.

# Common pitfalls
## Path sanitizing
Ruby has a bunch of File path related functions, but none of them is sufficient
for sanitizing a user inputted path to a developer-expecting-safe path in the
file system.

When dealing with file paths, developers usually want that all the files are
located inside a base directory, which could be a developer specified constant
or `Rails.root` Rails applications.

```
[1] pry(main)> ROOT='/home/nyangawa'
=> "/home/nyangawa"
```

### File::join
`join` is often used to concatenate paths without taking care of the trailing
slashes. But the user should be careful about the following conditions:

```
[2] pry(main)> File.join(ROOT, '../../etc/passwd')
=> "/home/nyangawa/../../etc/passwd"
```

Here the path string still begins with `/home/nyangawa` while the actual file
it directs to is actually `/etc/passwd`

So, before joining any user input with your `ROOT`, be sure to sanitize the string
first and check the pattern `..`

### File::expand_path
`expand_path` is also a method which I noticed to be used by many developers for
path concatenation as this method can "expand" the path and get rid of some
meaningless patterns like `././././` which `File::join` does not handle, and
finally returns the absolute path of the concatenated path in the file system.

```
[3] pry(main)> File.expand_path('dir/./././file', ROOT)
=> "/home/nyangawa/dir/file"
```

However, even if the input is checked and ensured that no `..` included. This
function can still be used to bring some surprises.

```
[4] pry(main)> File.expand_path('/etc/passwd', ROOT)
=> "/etc/passwd"

[5] pry(main)> File.expand_path('~sys', ROOT)
=> "/dev"
```

Reading the document of `expand_path` can help to understand the behavior.

https://ruby-doc.org/core-2.5.3/File.html#method-c-expand_path


So the general recommendation is, no matter what function you use to concatenate
user input to your base path, comparing the result to the base path with string
method like `start_with?` would be a good idea.

But is that enough?

## Symbolic links
The answer is probably no due to the existence of symbolic links. A symbolic
links could be used to escape out of the base directory without showing any
differences on its path with any other normal files.

```
[1] pry(main)> File.symlink('/etc/passwd', 'linkfile')
=> 0
[2] pry(main)> File.read('linkfile')
=> "root:x:0:0:root:/root:/bin/bash\ndaemon..."
```

Therefore, before reading any file which might be controllable or partially
controllable by external users, we should check the file first. And there are a
couple of useful methods for this purpose.

### File::realpath

`File::realpath` is a good example, it follows a symbolic link to its real
destination and returns the absolute path so that it could be checked against
our expected base directory.

```
[1] pry(main)> File.realpath('linkfile')
=> "/etc/passwd"
```

### File#lstat

If you simply want to deny the usage of symbolic links to avoid any unexpected
occasions. Using `File#lstat` to check the file stats is my recommendation, if
it is a symbolic link, exit loudly.

```
[1] pry(main)> File.new('linkfile').lstat.symlink?
=> true
```

Be noticed `File#stat` can't recognize a link because it follows it to the
directed file.

```
[1] pry(main)> File.stat('linkfile').symlink?
=> false
```


# TBC

```
## File path glob
## Command Injections
## Neglected dangerous functions

# How to find these bugs from existing code
```
