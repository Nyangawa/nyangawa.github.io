---
title:  "Secure coding with Ruby from the perspective of a system engineer (2)"
date:   2018-10-31 16:55:15 +0800
categories: security
---

# Before I continue

For some of the pitfalls I mentioned in last post of this series, there were
actually some real-world bugs caused by invalid checks to file paths, not only
CVE-2018-14364, which I wrote a blog to explain months ago. I'll reveal them in
near future.

I was planning to write more about the common pitfalls at the end of the last
post. Including glob, command injections and some often neglected dangerous
functions, plus several network-related issues like IPv6. But I've changed my
mind, I'm going to jump to the last part about how to find the vulnerable code
snippets in a could be large code base, effectively. And then jump back to the
pitfalls and verify the method to see how effective it could be.

# How to search code

Source code auditing could be fun and beneficial at first, I learned a lot from
the source code of GitLab and GitHub Enterprise and some other well-known
projects. Through auditing their code from the very beginning to the end, I
could see how they solve problems and how they create new mechanisms to protect
their applications.

However, manual code auditing is time consuming. And spending weeks on it, I
began to feel tired to look over similar code snippets, and this condition made
me neglect probably valuable parts of the code. So I started to consider that
whether and how to find a way to automate the most time consuming part of code
auditing.

# String matching

I used `ag` and `grep` to search in the code base, these tools are great for
repositories in small scale. When you want to search a call to a specific
function, and the target function has a considerably unique name. It could be as
easy as the following search:

``` shell
~/projects/gitlab-ce$ ag 'remove_symlinks'
lib/gitlab/import_export/file_importer.rb
23:        remove_symlinks
34:        remove_symlinks
66:      def remove_symlinks
```

the target function `remove_symlinks` only occurs in this file. The string
match tool works perfectly here.

## methods have same name

But `ag` doesn't always work as we wish like that. The `remove_symlinks` is
called in `Gitlab::ImportExport::FileImporter#import`, which is a public
instance method.

``` shell
      def import
        mkdir_p(@shared.export_path)
        mkdir_p(@shared.archive_path)

        remove_symlinks
        copy_archive
...
      ensure
        remove_import_file
        remove_symlinks
      end
```

What if we want to see who invokes this `import` function now?

``` shell
$ grep import . -r | wc -l
13649
```

Word `import` occurs more than 10 thousand times in the code base.

``` shell
$ fgrep '.import' . -r | wc -l
481
```

Even if we optimize the pattern, hundreds of lines of code is still not that
acceptable.

And more importantly,

``` shell
$ egrep 'def import$' . -r
./app/controllers/projects/project_members_controller.rb:  def import
./lib/gitlab/import_export/file_importer.rb:      def import
```

as a public instance method name, it's almost impossible for a string
matching tool to tell that `project_members_controller_instance.import`
and `file_importer_instance.import` are totally different functions. Therefore
that is really hard to filter out those we don't want to see.

## Dynamic (tainted) string
Methods that have same name in different classes is not the only problem.
Consider we want to find all occurrences of `eval` with a potentially
controllable argument, like `eval(params[:code])`. But for those invocations
like `eval('1+1')`, the argument is static and not controllable at all, so I
don't want this to be output to the final result.

Someone might say, "that's really easy, use /eval\(["|']/". But what about the
following cases:

``` ruby
eval params[:code] # Ruby allows function calls without parentheses
eval "result=#{params[:code]}" # String interpolation
eval("result=" + params[:code]) # String concatenation
eval \
params[:code] # One function call in multiple lines
...
```

I'm not a master of regular expressions, and I'm not going to be as it will
probably cost more time for me to write and test the regular expressions than
I do some manual auditing work as before.

# AST matching

The problem of string matching tools is that they don't really understand the
code. Tools like `grep` expect the user to tell them, for example, what's the
pattern of a function call and which variable is of expected class and which is
not. So, a string matching approach highly depends on the understanding and
experience of the human who writes the pattern. I believe @matz has the ability
to write some reliable regular expressions to solve the above questions, but
he probably will not do this due to the existence of Ruby AST.

FYI, AST is for 'Abstract Syntax Tree'

## Rubocop

So, the point is, we need something who really understand the source code to
read the code first, then process it to general lexical structures for the
matching part to output the final result.

Fortunately, [Rubocop](https://github.com/rubocop-hq/rubocop) is an open-source
project which has done splendid work on static analyzing of Ruby. And it
provides an extensive interface for users to create their own lexical matching
rules.

The core of AST matching in Rubocop is '[node pattern](https://docs.rubocop.org/en/latest/node_pattern/)'.
User can write a expected AST pattern and let Rubocop to find all matching
patterns and output the location of the corresponding source code.

For example, to match a 'dangerous (probably)' call of `eval`, I can define a
pattern like:

``` text
(send {nil? (const nil? :Kernel} :eval (!str ...))
```

pattern like this will match every direct call of `eval` or `Kernel.eval` where
the argument is NOT a static string.

The output would be like
``` text
$ rubocop -r ./cops/tainted_eval.rb --only Custom test/test_tainted_eval.rb
Inspecting 1 file
C

Offenses:

test/test_tainted_eval.rb:5:1: C: Custom/TaintedEval: Found eval with tainted input
Kernel.eval var
^^^^^^^^^^^^^^^
test/test_tainted_eval.rb:6:1: C: Custom/TaintedEval: Found eval with tainted input
eval var
^^^^^^^^
test/test_tainted_eval.rb:7:1: C: Custom/TaintedEval: Found eval with tainted input
eval "test#{var}"
^^^^^^^^^^^^^^^^^
test/test_tainted_eval.rb:8:1: C: Custom/TaintedEval: Found eval with tainted input
eval \ ...
^^^^^^

1 file inspected, 4 offenses detected
```

where the file I used for test is:

``` ruby
# test/test_tainted_eval.rb
# safe
eval 'string_literal'

# vul
Kernel.eval var
eval var
eval "test#{var}"
eval \
  'result=' + @var
```

It successfully filters out the safe `eval` with a string literal and tells me
that the left calls are probably dangerous with where they are.

Of course, the example above is just a trivial example while many causes are not
considered. But what I want to express here is that with the static analysis
approach, we can save a lot of time in code auditing by defining some
well-designed patterns, without losing too many valuable parts and receiving
also too many noises like we used to do with a string matching tool.

In the next part of this series, I'm going to show some examples about how
Rubocop with custom cops was used during my latest code auditing trip. And It
helped me to finally identified a remote code execution bug in Gitlab(CVE-2018-18649).
