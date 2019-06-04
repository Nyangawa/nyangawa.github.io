---
title:  "Arbitrary data deserialization in Ruby"
date:   2018-04-08 13:44:11 +0800
categories: security
---
Deserializing untrusted data is usually dangerous and can lead to serious
consequences like remote code execution. Remember not to deserialize user-input
data unless you really know what you are doing.

Ruby provides several ways of serializing/deserializing an object in ruby-core
and ruby-stdlib. The most commonly used are `Marshal` and `YAML` (`json` also
has the ability of processing objects in ruby but this will be another story)

The story starts from [CVE-2013-0156](https://www.cvedetails.com/cve/cve-2013-0156)
in Ruby on Rails, where `YAML.load` could be called with arbitrary data when the
framework processes an HTTP request. There're already some great
[write-ups](https://github.com/charliesome/charlie.bz/blob/master/posts/rails-3.2.10-remote-code-execution.md)
on this issue so I'm not going to repeat the analysis of the payload.

Generally speaking, the exploits are based on a similar way:

1. Find a call of `YAML.load`(or `Marshal.load`) where the first parameter
could be controlled.
2. Find a class with a "dangerous method" which will be used to execute the
real payload afterwards (the trojan).
3. Forge the object of our target class with required properties set as expected
to complete the execution chain.
4. `dump` (`Marshal` or `YAML`) the above object to bytes and send it to the
target.

The descriptions above might be obscure for who didn't look into the issue
carefully. I'll take `ERB` as an example since it exactly fullfills the
requirements of a "deserialization trojan".

{% highlight ruby %}
/usr/lib/ruby/2.5.0/erb.rb
  # Generate results and print them. (see ERB#result)
  def run(b=new_toplevel)
    print self.result(b)
  end

  #
  # Executes the generated ERB code to produce a completed template, returning
  # the results of that code.  (See ERB::new for details on how this process
  # can be affected by _safe_level_.)
  #
  # _b_ accepts a Binding object which is used to set the context of
  # code evaluation.
  #
  def result(b=new_toplevel)
    if @safe_level
      proc {
        $SAFE = @safe_level
        eval(@src, b, (@filename || '(erb)'), @lineno)
      }.call
    else
      eval(@src, b, (@filename || '(erb)'), @lineno)
    end
  end
{% endhighlight %}

A call to `ERB#result` or `ERB#run` will eventually lead to a `eval` where
the first parameter is the `@src` property. This is totally safe while the most
common use case of `ERB` is the load a template from local filesystem and render
it with a set of provided bindings.

But what will happen if we look into the following example:

{% highlight ruby %}

class Response
    attr_reader :result # let's say, for example, the wrapped response code
    ...
    ...
end

res = YAML.load(some_user_input)
puts "The result is #{res.result}" # To print the result

{% endhighlight %}

The `some_user_input` here actually expects the dump of an object of class Response.
But thanks to Ruby's duck typing, a dump of an `ERB` object is also eligible here.
Finally, the `ERB#result` will be called at the `puts` line instead of
`Response#result`. Therefore, in this case, forging the trojan is considerably easy.

{% highlight ruby %}

o = ERB.allocate
o.instance_variable_set :@src, "puts `id`"
o.instance_variable_set :@lineno, 1026
YAML.dump(o)

=> "--- !ruby/object:ERB\nsrc: puts `id`\nlineno: 1026\n"
{% endhighlight %}

## Conclusion
The above example is simplified as much as possible to make it clear enough to
understand. Cases in the real world are probably much more complicated to exploit
like [this one](https://justi.cz/security/2017/10/07/rubygems-org-rce.html).

Forging the invocation chain in the payload is usually the most tricky part.
During this procedure, some methods like `method_missing`, `send` or `public_send`
may worth a closer look.

Due to some historical reasons, the public resources about arbitrary data
deserialization are often in the scope of Ruby on Rails instead of Ruby the
language and its standard libraries. As Ruby on Rails and its dependencies
actually provide many gadgets for completing the invocation chain. However it
should be noticed that this issue also exists in common ruby projects. Although
the payload may look greatly different to those were used in RoR.
