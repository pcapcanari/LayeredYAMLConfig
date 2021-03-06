# LayeredYAMLConfig

home :: https://github.com/jf647/LayeredYAMLConfig

## SUMMARY:

Ruby configuration library that layers multiple YAML files on top of each
other with ERB evalution.

## DESCRIPTION:

LayeredYAMLConfig provides a simple config file that supports multiple
layers.  Values in the right or uppermost layers override values in lower
layers.  This makes it easy to share configuration without duplication while
still allowing what needs to be different to vary.

For example:

    program.default.conf
    program.server_foo.conf
    program.site_bar.conf
    program.conf

Optionally, leaf nodes can be evaluated using as ERB templates, feeding the
configuration into itself.

## Synopsis

```ruby
class MyConfig < LayeredYAMLConfig
end

MyConfig.skipbad = true
cfg = MyConfig.instance( 'ex7.yaml', 'ex8.yaml', 'ex9.yaml', 'ex10.yaml' )
puts cfg[:foo]['bar']
puts cfg['foo'][:gzonk]
```

ex7.yaml:
```yaml
---
foo:
    bar: baz
```

ex8.yaml:
```yaml
---
foo:
    gzonk: quux
```

ex10.yaml:
```text
This is not a YAML file
```

ex16.yaml:
```text
---
a: d
b: e
c: f
g:
    - <%= @cfg[:a] %>
    - <%= @cfg[:b] %>
    - <%= @cfg[:c] %>
```

To use LayeredYAMLConfig, create a new class that inherits from it.  The new
class is a singleton that can only be constructed the first time ::instance
is called.  Pass one or more YAML filenames which will be read and deep
merged on top of each other in left-to-right order.

Files that are missing are skipped by default.  Files that are bad (i.e. do
not parse as valid YAML) cause an exception to be thrown.  This behaviour
can be overridden by calling ::skipbad = true or ::skipmissing = true before
constructing the configuration.

The type of the returned object is your subordinate class, but #[] and #[]=
are delegated to the contained hash, which is an
ActiveSupport::HashWithIndifferentAccess.  This means you can use strings or
symbols interchangeably to access elements of the hash.

## Adding Layers after Instance Construction

Using #add, you can add one or more layers that are deep merged into the
existing config.

## Converting to a Hash

Call #to_hash to return a symbolized hash representation of the
configuration object

## ERB Templates

To enable template evaluation, call ::templates = true on the class:

```ruby
class OurConfig < LayeredYAMLConfig
end
OurConfig.templates = true
OurConfig.instance('file_containing_erb.yaml')
```

Every time #add is called (either directly or implicitly during creation),
the configuration tree is traversed.  Any String leaf nodes are evaluated
using Erubis.  The only context variable available is @cfg, which
representes the configuration at the start of the template pass.

The process of template evaluation is as follows:

1. Walk the tree, keeping track of how successful and failed template evaluations we performed
2. If there were no failures, the pass is complete
3. If there were no successes and the previous pass (if any) had at least one success, keep going
4. If there were no successes and the previous pass also had no successes, raise an exception
5. Keep going

Requiring two passes with no successful template evaluations allows files in
upper layers to depend on values defined in lower layers.  For example,
given these files:

```text
---
a:
    b:
        c: <%= @cfg[:d][:e][:f] %>

---
d:
    e:
        f: <%= @cfg[:g][:h][:i] %>

---
g:
    h:
        i: j
```

It would take two passes to fully resolve all templates.  In the first pass,
cfg[:d][:e][:f] would resolve to 'j', and in the second pass,
cfg[:a][:b][:c] would also resolve to 'j'.

### Empty Strings

If a template evaluates to the empty string, this is by default considered a
failure.  To treat empty strings as success, call ::emptyok before
constructing the configuration:

```ruby
class OurConfig < LayeredYAMLConfig
end
OurConfig.templates = true
OurConfig.emptyok = true
OurConfig.instance('file_containing_erb.yaml')
```

Beware of hidden gotchas though: if an intermediate node is undefined, ERB
will throw an exception trying to deference nil.  But if only the last node
is defined, then ERB will generate an empty string:

```text
---
a:
    b:
        c: d
        e: <%= @cfg[:a][:b][:f] %>
        g: <%= @cfg[:a][:h][:i] %>
```

In the default mode, neither template resolves successfully.  With ::emptyok
enabled, cfg[:a][:b][:e] becomes the empty string.

### What can be expanded

The result of an Erubis expansion is always a string.  Therefore, you can't
expand a Hash or Array and then expect it to dereference properly.  This,
for example, won't work:

```text
a:
    one: 1
    two: 2
    three: 3
b: <%= @cfg[:a] %>
```

```ruby
puts cfg[:b][:one]
```

The expansion of cfg[:b] isn't the same as cfg[:a], it's the same as
cfg[:a].to_s.

You can however dereference through a Hash or Array to a scalar leaf:

```text
a:
    one: 1
    two: 2
    three: 3
b: <%= @cfg[:a][:one] %>
```

```ruby
puts cfg[:b]
```

## Resetting Per-Class Options

call ::reset on your class to reset the ::skipbad, ::skipmissing,
::templates and ::emptyok settings to their defaults.  Call ::reset_all on
the base class to reset these options to default for all derived classes.

## LICENSE:

The MIT License (MIT)

Copyright (c) 2012, 2013 James FitzGibbon <james@nadt.net>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to
deal in the Software without restriction, including without limitation the
rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
sell copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.

## Contributing to LayeredYAMLConfig
 
* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it
* Fork the project
* Start a feature/bugfix branch
* Commit and push until you are happy with your contribution
* Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
* Please try not to mess with the version or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.
