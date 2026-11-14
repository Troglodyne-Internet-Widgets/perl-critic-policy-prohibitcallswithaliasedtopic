# NAME

Perl::Critic::Policy::Variables::ProhibitCallsWithAliasedTopic - Do not call a sub or method while $\_ is an alias to the caller's data.

# VERSION

version 0.001

# Perl::Critic::Policy::Variables::ProhibitCallsWithAliasedTopic

`map`, `grep`, `first` and the other list functions set `$_` to each
element of their list in turn, and so does a postfix `for`.  `$_` is then
not a copy.  It is an alias, so an assignment to `$_` changes the element.
`$_` is also one global variable.  So a sub that the block calls, and that
writes `$_` without a `local $_` of its own, writes the caller's element:

```perl
sub first_line {
    my ($file) = @_;
    open my $fh, '<', $file or die "$file: $!";
    while (<$fh>) { last }     # assigns each line to $_
    return $_;
}

my @heads = map { first_line($_) } @files;    # reported
# @files now holds undef, or the first line of each file
```

The writer is seldom in sight.  It is a `while (<$fh`)>, or a bare `chomp;`
or `s///`, in a helper sub, often in another module.  Nothing in the block
looks wrong.

In a list function block, say `local ($_) = $_;` before the first call.  The
block then has a `$_` of its own, with the same value, and a callee that
writes `$_` writes that copy:

```perl
my @heads = map { local ($_) = $_; first_line($_) } @files;    # OK!
```

A postfix `for` has no block to put a `local` in, and it cannot name its
variable.  Write it as a block `for` with a lexical variable, which does not
make `$_` an alias at all:

```perl
$_->load for @documents;                  # reported
for my $doc (@documents) { $doc->load }   # OK!
```

## PROHIBITED

Each of these calls, in the block of a list function before any
`local $_`, or before the `for` of a statement with a postfix `for`:

```perl
f($_)                     # a sub
Some::Where::f($_)        # a sub by its full name
$_->is_ok                 # a method
Some::Class->new($_)      # a class method
$o->$name                 # a method by its name in a variable
$code->($_)               # a code reference
&$code  &f                # with an ampersand
PI                        # a constant is a sub too
```

A copy does not help: the callee still writes `$_`, which is still the
element.

```perl
my @r = map { my $y = $_; f($y) } @x;                 # reported
```

`map EXPR, LIST` and `grep EXPR, LIST` alias `$_` as the block forms do:

```perl
my @r = map f($_), @x;                                # reported
```

## ALLOWED

```perl
my @r = map { lc } @x;                       # a builtin is not a call
my @r = map { local ($_) = $_; f() } @x;     # the local comes first
my @r = map { my $y = $_; local $_; f($y) } @x;
sub f { ... } my @r = map { f($_) } @x;      # see below
my @r = map { $_ } f(@x);                    # the list is made first
my @r = map { sub { f() } } @x;              # not called here
my @r = sort { f($a) <=> f($b) } @x;         # sort does not set $_
print for @x;                                # no call
for my $y (@x) { f($y) }                     # no alias
```

In a list function block, a call of a sub that is defined in the same file, or
in another file of the same distribution, is left alone.
[Perl::Critic::Policy::ControlStructures::ProhibitMutatingListFunctionsStricter](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AControlStructures%3A%3AProhibitMutatingListFunctionsStricter)
reads the body of that sub, and reports the block if the sub changes `$_`.
So the two policies can be enabled together, and neither reports what the
other does.  The name is matched without its package, so a sub named `trim`
anywhere in the distribution covers every call of `trim`.  A distribution is
what [Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution) finds above the file; a snippet with no file
has only its own subs.  A method call is never left alone, because the method
that runs can be another sub of the same name.  A postfix `for` has no such
policy, so a call of any sub counts there.

## WHICH CONSTRUCTS

A list function block is the block of `map`, `grep`, and of each list
function that makes `$_` an alias to its elements, by its full name or with
no package.  These were each checked on perl 5.44, with a callee that assigns
to `$_`:

```
List::Util       first any all none notall
List::MoreUtils  any all none notall true false firstidx first_index
List::SomeUtils  lastidx last_index firstval first_value lastval
                 last_value firstres first_result lastres last_result
                 onlyidx only_index onlyval only_value onlyres
                 only_result indexes after after_incl before
                 before_incl part insert_after
List::UtilsBy    max_by min_by
```

These do not alias `$_`, and are not list functions here: `sort` and
`reduce`, which set `$a` and `$b`; `apply` of List::MoreUtils, and
`sort_by`, `uniq_by` and the other functions of List::UtilsBy, which give
the block a copy.

Inside the block, `$_` belongs to another construct in three places, and a
call there is not this block's: the body of a `foreach` with no loop variable,
which saves `$_` and puts it back; the block of a list function, or the part
of a statement before a postfix `for`, which are reported on their own; and
an anonymous sub, which runs when it is called.

A block `foreach` with no loop variable makes `$_` an alias as well, but it
can name one.  [Perl::Critic::Policy::Variables::ProhibitTopicIterator](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AVariables%3A%3AProhibitTopicIterator), in
Perl-Critic-Grape, reports each of them, and this policy does not.

A `local $_` covers the calls in the statements after it, at the top level of
the block.  It does not cover a call in the `local` statement itself:
`local ($_) = f($_)` calls `f` before `$_` is localized.  It can be
`local $_;`, `local $_ = $_;` or `local ($_) = $_;`.  Of those, only the
last passes both
[Perl::Critic::Policy::ControlStructures::ProhibitMutatingListFunctions](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AControlStructures%3A%3AProhibitMutatingListFunctions),
which takes `local $_ = $_` for a change to `$_`, and
[Perl::Critic::Policy::Variables::RequireInitializationForLocalVars](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AVariables%3A%3ARequireInitializationForLocalVars), which
reports `local $_;`.  After `local ($_) = $_;` the block reads the copy, so
a block that wants the value from before the call copies it into a lexical
first.

## CAVEATS

A call that perl makes for the code, and that the code does not spell, is not
seen: an overloaded operator, a tied variable, a `DESTROY`, `sort SUBNAME`,
`require` and `do FILE`.

Some lists are not the caller's data, and the policy reports a call there all
the same.  `keys` returns copies, so a write to `$_` in
`for (keys %h)` changes nothing that lasts.  A write to an element of a
literal list, as in `for (1, 2)`, dies with "Modification of a read-only value
attempted", so it does not pass silently.  `values` does return aliases.

# CONFIGURATION

`allow` names subs and methods that are known not to write `$_`.  A sub is
matched as it is written in the call, so `Some::g` does not allow `g`, and a
method by its name alone.  The default is `can DOES isa VERSION`, the methods
of `UNIVERSAL`, which perl provides.  A list replaces the default, so name
them too to keep them:

```
[Variables::ProhibitCallsWithAliasedTopic]
allow = can DOES isa VERSION to_string Some::Util::is_blank id name
```

`list_funcs` replaces the list functions above, and `add_list_funcs` adds to
them:

```
[Variables::ProhibitCallsWithAliasedTopic]
add_list_funcs = My::Util::each_item each_item
```

## METHODS

### supported\_parameters

### default\_severity

### default\_themes

### applies\_to

### initialize\_if\_enabled

Joins the list functions of the two parameters, and registers with
[Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution) what this policy needs from each file of a
distribution: the names of its subs, without their package.

### violates

### known\_subs

The names, without their package, of the subs of a document and of the rest
of its distribution.  Worked out once for each document, since `violates` is
called once for each word.

## FUNCTIONS

### sub\_names\_in

The names of the named subs of a [PPI::Document](https://metacpan.org/pod/PPI%3A%3ADocument) or
[Perl::Critic::Document](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADocument), lexical ones too, without their package.

### bare\_name

The part of a name after its last `::`, which is all of a name with no package.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/teodesian/perl-critic-policy-prohibitcallswithaliasedtopic/issues](https://github.com/teodesian/perl-critic-policy-prohibitcallswithaliasedtopic/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# CONTRIBUTOR

troglodyne-bot <morpheus@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
