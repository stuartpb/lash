# lash

A simply shellish scripting language

To be clear, none of this is *actually implemented*. The text below is just a statement of intent. See also [this Twitter exchange](https://twitter.com/stuartpb/status/733156374830415877) with @tj.

## How is Lash different from (bash/zsh/csh/fish/lish/lush/pish/posh/shsh)?

Unlike most other attempts to rethink shell scripting, Lash **does not attempt to resemble traditional shell languages**, in either its syntax or its underlying model.

Most attempts to write a "modern" shell have been focused on keeping the *terse syntax* of a traditional shell language, and adding *features from other languages* to it - sometimes even neglecting to retain basic features of older shells, like job control or I/O redirection. In terms of goals, this is like making a garbage truck nicer by putting seating in the garbage, and replacing the steering wheel with a live kitten.

Lash does not aim to make common operations *prettier*: it aims to make primitive concepts *composable*, and in doing so, make *complex* operations *possible*. It does this by starting from the [Lua][] scripting language, then adding a few facilities to make it capable of accomplishing the same tasks as Bash (along with a *little* bit of syntactic sugar to decrease the verbosity of basic task running).

[Lua]: https://www.lua.org

## How does Lash diverge from Lua?

### Added functions

Everything from `os` and `io` are dumped into `_G`, since `os` and `io` calls are the entire point of the language.

There are also `proc`, `buffer`, and `pipe` tables,  along with the `qarglist` function:

#### proc.spawn(name, ...)

This is also the `_call` metamethod of `proc`.

May take extended options, for things that need to be associated on spawn (like environment variables or file descriptors).

The default values for file descriptors when spawning will probably be `nil`, meaning they'll be directed to `/dev/null`, instead of inheriting the process's file descriptors (since that's weird and leaky).

The returned process object has `proc` as its `__index` metamethod, like how strings have `string` as their prototype.

#### proc.wait(process)

Blocks until the given process has terminated. Returns the exit code.

#### buffer.new()

This is also the `_call` metamethod of `buffer`.

Buffers work like in-memory files that can be handled like streams.

#### buffer.drain(buf, target)

Pipes the given `buf` to `target`, and clears the buffer as it is read by `target`. (This allows a buffer to be used essentially like a pipe with indefinite backpressure.)

#### buffer.finish(buf)

Returns the (remaining) contents of the buffer as a string and empties / deletes it.

#### pipe.new()

Creates an anonymous pipe for streaming output to something's input.

#### qarglist(str)

Parses `str` according to roughly the same rules as would be used by `xargs` and returns the table of the split strings specified by that string.

### Removals

- The `userdata` type is replaced with a `buffer` type.
- The ability to load C libraries is removed.

I'm not sure about this part, actually. Removing this stuff *would* make Lash not a superset of Lua, which I feel would make Lash cleaner (as languages that are just Lua-with-extensions-bolted-on are gross). However, the only real benefits of that are not making the language subject to ABI breakage in future versions, and cutting it off from the Lua ecosystem, neither of which I'm convinced are advantages over the ability to interface with C modules or incorporate Lua code.

### The added syntax

Since unpacking is a really common construction (just like getting the length of a table), `table.unpack` gets a language-level operator in Lash (just like `table.maxn` got `#` in Lua 5.1):

```
arglist... -- equivalent to table.unpack(arglist)
```

The `$` operator provides syntactic sugar for spawning a process, with shell-style string specification. It is terminated by an end-of-line, semicolon, or closing parenthesis (matching an open parenthesis preceding it), and can contain embedded Lash expressions within matched parentheses:

```
$ foo bar (baz, zar) "quux\n"(j)'stuff'
```

is equivalent to something like (since all the details are in the air right now):

```
return proc.spawn('foo', {fds = {[0] = stdin, stdout, stderr}},
  'bar', baz, zar, "quux\n"..j..'stuff'):wait()
```

Single-quoted strings take everything up to the next single quote literally. Backslash escaping outside of single-quotes works as they do for regular Lua single/double-quoted string syntax (all non-letter characters are escapes, some letter characters like `\n` are special). Arguments are separated by spans of whitespace. Everything not separated by unquoted space is considered part of one string. Quoted strings may contain newlines (as well as other forms of whitespace). Backslashes can escape semicolons or newlines (escaped unquoted newlines are treated as argument-separating whitespace unless escaped by a `\z`, where they will be ignored - an encoded newline, outside of a quoted string, would have to be preceded by `\n\z`, to literally specify a newline and then ignore the literal newline as separating space).

Parenthesized expressions isolated from any concatenating values (ie. with whitespace on both sides) may return multiple values to include multiple arguments. (Expressions that may return multiple values that intend to represent only one argument value should use a second pair of parentheses within the expression, eg. `((string.gsub(foo,'%w','A')))`.) 

Parenthesized expressions placed adjacent to a concatenating value without argument-separating space (eg. `(foo)bar`) are treated as an error if returning anything other than a single string or number, rather than discarding subsequent values, applying `tostring`, defining multiple arguments, and/or concatenting adjacent strings to the first/last/every value. Any of these behaviors, if desired, can be achieved by performing the applicable transformation within the context of the parenthesized expression.

There is **no** `$expansion`, even in quoted strings. The *only* place where the text that's inside a quoted string is interpreted as something other than itself is an escape character preceded by a backslash. If you want an expression as part of a quoted string, you have to end the quoting and use parentheses (or, really, just don't use quotes at all - since Lash pretty much only gives printing characters a non-literal meaning within embedded parenthetical expressions, quotes are really only called for in the rare situation where your strings embed spaces and/or parentheses).

Wildcard characters only apply outside of *any* quoting, and even then, again, I'm not sure I want it to apply. (Also, if they *are* implemented, it'll be as nullglobs, where no matches are evaluated to no strings.) I think globbing would probably be better handled with an explicit function call (see note above about Lash being meant to have almost no non-literal characters outside of quoting beond spaces and parentheses).

The `>` and `<` operators (specified *before* the command), in addition to accepting filenames, accept buffers, file descriptors, pipe objects, `nil` (as a standin for `/dev/null`) etc via `()`:

```
local logs = buffer.new()
$ <(logs) logging-command
$ >(logs) foo bar (baz, zar) "quux\n$j"'stuff'
$ >(logs) another-command like magic
```

```
local logs = buffer.new()
proc.spawn('logging-command', {fds = {[0] = stdin, logs, stderr}}):wait()
proc.spawn('foo', {fds = {[0] = logs, stdout, stderr}},
  'bar', baz, zar, "quux\n"..j..'stuff'):wait()
```

File descriptors in embedded argument expression values are coerced to the equivalent `/dev/fdX` name, with buffers, anonymous pipes, and process handles (from `proc.spawn()`) attaching and coercing to the appropriate fd63-and-up descriptors (akin to how Bash does process substitution). These *may* be concatenated with strings, for the purpose of named options which take file descriptors, like `--output=(logfile)`.

```
local imbuf = buffer() -- since this is the same thing as buffer.new()
local fullsize = open('cat.jpg','w');
$ curl -L http://thecatapi.com/api/images/get?format=jpg -o (imbuf)
fullsize:write(imbuf):close();
$ convert (imbuf) -thumbnail 240x240 catthumb.jpg
```

This will probably be surfaced with some kind of function like `fdify` (or whatever the actual corresponding syscall is called), which `proc.spawn` will also apply if receiving one of these attachable types as an argument value.

Variants:

`$$` returns the output of the command as a string, like command substitution:

```
local b = buffer.new()
proc.spawn('foo', {fds = {[0] = stdin, b, stderr}},
  'bar', baz, zar, "quux\n"..j..'stuff'):wait()
return b:finish()
```

`$&` returns just the process handle:

```
return proc.spawn('foo', {fds = {[0] = stdin, stdout, stderr}},
  'bar', baz, zar, "quux\n"..j..'stuff')
```
