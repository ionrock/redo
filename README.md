# redo: a top-down software build system

`redo` is a competitor to the long-lived, but sadly imperfect, `make`
program.  There are many such competitors, because many people over the
years have been dissatisfied with make's limitations.  However, of all the
replacements I've seen, only redo captures the essential simplicity and
flexibility of make, while avoiding its flaws.  To my great surprise, it
manages to do this while being simultaneously simpler than make, more
flexible than make, and more powerful than make.

Although I wrote redo and I would love to take credit for it, the magical
simplicity and flexibility comes because I copied verbatim a design by
Daniel J. Bernstein (creator of qmail and djbdns, among many other useful
things).  He posted some very terse notes on his web site at one point
(there is no date) with the unassuming title, "[Rebuilding target files when
source files have changed](http://cr.yp.to/redo.html)." Those notes are
enough information to understand how the system it supposed to work;
unfortunately there's no code to go with it.  I get the impression that the
hypothetical "djb redo" is incomplete and Bernstein doesn't yet consider it
ready for the real world.

I was led to that particular page by random chance from a link on [The djb
way](http://thedjbway.b0llix.net/future.html), by Wayne Marshall.

After I found out about djb redo, I searched the Internet for any sign that
other people had discovered what I had: a hidden, unimplemented gem of
brilliant code design.  I found only one interesting link: Alan Grosskurth,
whose [Master's thesis at the University of Waterloo](http://grosskurth.ca/papers/mmath-thesis.pdf)
was about top-down software rebuilding, that is, djb redo.  He wrote his
own (admittedly slow) implementation in about 250 lines of shell script.

If you've ever thought about rewriting GNU make from scratch, the idea of
doing it in 250 lines of shell script probably didn't occur to you.  redo is
so simple that it's actually possible.  For testing, I actually wrote an
even more minimal version,which always rebuilds everything instead of
checking dependencies, in 100 lines of shell (less than 2 kbytes).

The design is simply that good.

My implementation of redo is called `redo` for the same reason that there
are 75 different versions of `make` that are all called `make`.  It's somehow
easier that way.  Hopefully it will turn out to be compatible with the other
implementations, should there be any.

My extremely minimal implementation, called `do`, is in the `minimal/`
directory of this repository.


# License

My version of redo was written without ever seeing redo code by Bernstein or
Grosskurth, so I own the entire copyright.  It's distributed under the GNU
LGPL version 2.  You can find a copy of it in the file called LICENSE.


# What's so special about redo?

The theory behind redo is almost magical: it can do everything `make` can
do, only the implementation is vastly simpler, the syntax is cleaner, and you
can do even more flexible things without resorting to ugly hacks.  Also, you
get all the speed of non-recursive `make` (only check dependencies once per
run) combined with all the cleanliness of recursive `make` (you don't have
code from one module stomping on code from another module).

(Disclaimer: my current implementation is not as fast as `make` for some
things, because it's written in python.  Eventually I'll rewrite it an C and
it'll be very, very fast.)

The easiest way to show it is with an example.

Create a file called default.o.do:

	redo-ifchange $1.c
	gcc -MD -MF $3.deps.tmp -c -o $3 $1.c
	DEPS=$(sed -e "s/^$3://" -e 's/\\//g' <$3.deps.tmp)
	rm -f $3.deps.tmp
	redo-ifchange $DEPS

Create a file called myprog.do:

	DEPS="a.o b.o"
	redo-ifchange $DEPS
	gcc -o $3 $DEPS
	
Of course, you'll also have to create `a.c` and `b.c`, the C language
source files that you want to build to create your application.

In a.c:

	#include <stdio.h>
	#include "b.h"
	
	int main() { printf(bstr); }
	
In b.h:

	extern char *bstr;
	
In b.c:
	char *bstr = "hello, world!\n";

Now you simply run:

	$ redo myprog
	
And it says:

	redo  myprog
	redo    a.o
	redo    b.o

Now try this:

	$ touch b.h
	$ redo myprog
	
Sure enough, it says:

	redo  myprog
	redo    a.o

Did you catch the shell incantation in `default.o.do` where it generates
the autodependencies?  The filename `default.o.do` means "run this script to
generate a .o file unless there's a more specific whatever.o.do script that
applies."

The key thing to understand about redo is that declaring a dependency is just
another shell command.  The `redo-ifchange` command means, "build each of my
arguments.  If any of them or their dependencies ever change, then I need to
run the *current script* over again."

Dependencies are tracked in a persistent `.redo` database so that redo can
check them later.  If a file needs to be rebuilt, it re-executes the
`whatever.do` script and regenerates the dependencies.  If a file doesn't
need to be rebuilt, redo can calculate that just using its persistent
`.redo` database, without re-running the script.  And it can do that check
just once right at the start of your project build.

But best of all, as you can see in `default.o.do`, you can declare a
dependency *after* building the program.  In C, you get your best dependency
information by trying to actually build, since that's how you find out which
headers you need.  redo is based on the following simple insight:
you don't actually
care what the dependencies are *before* you build the target; if the target
doesn't exist, you obviously need to build it.  Then, the build script
itself can provide the dependency information however it wants; unlike in
`make`, you don't need a special dependency syntax at all.  You can even
declare some of your dependencies after building, which makes C-style
autodependencies much simpler.

(GNU make supports putting some of your dependencies in include files, and
auto-reloading those include files if they change.  But this is very
confusing - the program flow through a Makefile is hard to trace already,
and even harder if it restarts randomly from the beginning when a file
changes.  With redo, you can just read the script from top to bottom.  A
`redo-ifchange` call is like calling a function, which you can also read
from top to bottom.)


# Does it make cross-platform builds easier?

A lot of build systems that try to replace make do it by
trying to provide a lot of predefined rules.  For example,
one build system I know includes default rules that can build C++
programs on Visual C++ or gcc, cross-compiled or not
cross-compiled, and so on.  Other build systems are
specific to ruby programs, or python programs, or Java or .Net
programs.

redo isn't like those programs; it's more like make.  It
doesn't know anything about your system or the language
your program is written in.

The good news is: redo will work with *any* programming
language with about equal difficulty.  The bad news is:
you might have to fill in more details than you would if you
just use ANT to compile a Java program.

So the short version is: cross-platform builds are about
equally easy in make and redo.  It's not any easier, but
it's not any harder.

FIXME:
Tools like automake are really just collections of Makefile
rules so you don't have to write the same ones over and
over.  In theory, someone could write an automake-like tool
for redo, and you could use that.


# Hey, does redo even *run* on Windows?

FIXME:
Probably under cygwin.  But it hasn't been tested, so no.

If I were going to port redo to Windows in a "native" way,
I might grab the source code to a posix shell (like the
one in MSYS) and link it directly into redo.

`make` also doesn't *really* run on Windows (unless you use
MSYS or Cygwin or something like that).  There are versions
of make that do - like Microsoft's version - but their
syntax is horrifically different from one vendor to
another, so you might as well just be writing for a
vendor-specific tool.

At least redo is simple enough that, theoretically, one
day, I can imagine it being cross platform.


# One script per file?  Can't I just put it all in one big Redofile like make does?

One of my favourite features of redo is that it doesn't add any new syntax;
the syntax of redo is *exactly* the syntax of sh... because sh is the program
interpreting your .do file.

Also, it's surprisingly useful to have each build script in its own file;
that way, you can declare a dependency on just that one build script instead
of the entire Makefile, and you won't have to rebuild everything just
because of a one-line Makefile change.  (Some build tools avoid that same
problem by tracking which variables and commands were used to do the build. 
But that's more complex, more error prone, and slower.)

Still, it would be rather easy to make a "Redofile" parser that just has a
bunch of sections like this:

	myprog:
		DEPS="a.o b.o"
	        redo-ifchange $DEPS
                gcc -o $3 $DEPS

We could just auto-extract myprog.do by slurping out the indented sections
into their own files.  You could even write a .do file to do it.

It's not obvious that this would be a real improvement however.

See djb's [Target files depend on build scripts](http://cr.yp.to/redo/honest-script.html)
article for more information.


# Can I set my dircolors to highlight .do files?

Yes!  At first, having a bunch of .do files in each
directory feels like a bit of a nuisance, but once you get
used to it, it's actually pretty convenient; a simple 'ls'
will show you which things you might want to redo in any
given directory.

Here's a chunk of my .dircolors.conf:

	.do 00;35
	*Makefile 00;35
	.o 00;30;1
	.pyc 00;30;1
	*~ 00;30;1
	.tmp 00;30;1

To activate it, you can add a line like this to your .bashrc:

	eval `dircolors $HOME/.dircolors.conf`


# What are the three parameters ($1, $2, $3) to a .do file?

$1 is the name of the target, with the extension removed,
if any.

$2 is the extension of the target, including the leading
dot.

$3 is the name of a temporary file that will be renamed to
the target filename atomically if your .do file returns a
zero (success) exit code.

In a file called `chicken.a.b.c.do` that builds a file called
`chicken.a.b.c`, $1 is `chicken.a.b.c`, $2 is blank, and $3 is a
temporary name like `chicken.a.b.c.tmp`.  You might have expected
$1 to be just `chicken`, but that's not possible, because
redo doesn't know which portion of the filename is the
"extension."  Is it `.c`, `.b.c`, or `.a.b.c`?

.do files starting with `default.` are special; they can
build any target ending with the given extension.  So let's
say we have a file named `default.c.do` building a file
called `chicken.a.b.c`. $1 is `chicken.a.b`, $2 is `.c`,
and $3 is a temporary name like `chicken.a.b.c.tmp`.

You should use $1 and $2 only in constructing input
filenames and dependencies; never modify the file named by
$1 in your script.  Only ever write to the file named by
$3.  That way redo can guarantee proper dependency
management and atomicity.  (For convenience, you can write
to stdout instead of $3 if you want.)

For example, you could compile a .c file into a .o file
like this, from a script named `default.o.do`:
	
	redo-ifchange $1.c
	gcc -o $3 -c $1.c

Note that $2, the output file's .o extension, is rarely useful
since you always know what it is.

FIXME: djb's design documentation doesn't clearly describe
$1 and $2, although it's clear that $3 is the output
filename.  We may have guessed $1 and $2, particularly $2,
incorrectly, so we might have to change their meanings
later in order to be compatible with djb's implementation.


# What happens to the stdin/stdout/stderr in a redo file?

As with make, stdin is not redirected.  You're probably
better off not using it, though, because especially with
parallel builds, it might not do anything useful.

As with make, stderr is also not redirected.  You can use
it to print status messages as your build proceeds.

Redo treats stdout specially: it redirects it to point at
$3 (see previous question).  That is, if your .do file
writes to stdout, then the data it writes ends up in the
output file.  Thus, a really simple `chicken.do` file that
contains only this:

    echo hello world

will correctly, and atomically, generate an output file
named `chicken` only if the echo command succeeds.


# Can a *.do file itself be generated as part of the build process?

Not currently.  There's nothing fundamentally preventing us from allowing
it.  However, it seems easier to reason about your build process if you
*aren't* auto-generating your build scripts on the fly.

This might change someday.


# Do end users have to have redo installed in order to build my project?

No.  We include a very short (99 lines, as of this writing) shell script
called `do` in the `minimal/` subdirectory of the redo project.  `do` is like
`redo` (and it works with the same `*.do` scripts), except it doesn't
understand dependencies; it just always rebuilds everything from the top.

You can include `do` with your program to make it so non-users of redo can
still build your program.  Someone who wants to hack on your program will
probably go crazy unless they have a copy of `redo` though.

Actually, `redo` itself isn't so big, so for large projects where it
matters, you could just include it with your project.


# How does redo store dependencies?

At the toplevel of your project, redo creates a directory
named `.redo`.  That directory contains a sqlite3 database
with dependency information.

The format of the `.redo` directory is undocumented because
it may change at any time.  If you really need to make a
tool that pokes around in there, please ask on the mailing
list if we can standardize something for you.


# If a target didn't change, how to I prevent dependents from being rebuilt?

For example, running ./configure creates a bunch of files including
config.h, and config.h might or might not change from one run to the next. 
We don't want to rebuild everything that depends on config.h if config.h is
identical.

With `make`, which makes build decisions based on timestamps, you would
simply have the ./configure script write to config.h.new, then only
overwrite config.h with that if the two files are different. 
However, that's a bit tedious.

With `redo`, there's an easier way.  You can have a
config.do script that looks like this:

	redo-ifchange autogen.sh *.ac
	./autogen.sh
	./configure
	cat config.h configure Makefile | redo-stamp
	
Now any of your other .do files can depend on a target called
`config`.  `config` gets rebuilt automatically if any of
your autoconf input files are changed (or if someone does
`redo config` to force it).  But because of the call to
redo-stamp, `config` is only considered to have changed if
the contents of config.h, configure, or Makefile are
different than they were before.

(Note that you might actually want to break this .do up into a
few phases: for example, one that runs aclocal, one that
runs autoconf, and one that runs ./configure.  That way
your build can always do the minimum amount of work
necessary.)


# Why not always use checksum-based dependencies instead of timestamps?

Some build systems keep a checksum of target files and rebuild dependents
only when the target changes.  This is appealing in some cases; for example,
with ./configure generating config.h, it could just go ahead and generate
config.h; the build system would be smart enough to rebuild or not rebuild
dependencies automatically.  This keeps build scripts simple and gets rid of
the need for people to re-implement file comparison over and over in every
project or for multiple files in the same project.

There are disadvantages to using checksums for everything,
however:

- calculating checksums for every output file adds time to
  the build;
  
- it makes it hard to *force* things to rebuild when you
  know you absolutely want that;
  
- targets that are just used for aggregation (ie. they
  don't produce any output of their own) would always have
  the same checksum - the checksum of a zero-byte file -
  which causes confusing results.

Thus, we made the decision to only use checksums for
targets that explicitly call `redo-stamp` (see previous
question).


# Why does 'redo target' always redo the target, even if it's unchanged?

When you run `make target`, make first checks the
dependencies of target; if they've changed, then it
rebuilds target.  Otherwise it does nothing.

redo is a little different.  It splits the build into two
steps.  `redo target` is the second step; if you run that
at the command line, it just runs the .do file, whether it
needs it or not.

If you really want to only rebuild targets that have
changed, you can run `redo-ifchange target` instead.


# Can my .do files be written in a language other than sh?

Yes.  If the first line of your .do file starts with the
magic "#!/" sequence (eg. `#!/usr/bin/python`), then redo
will execute your script using that particular interpreter.

Note that this is slightly different from normal Unix
execution semantics: redo never execs your script directly;
it only looks for the "#!/" line.  The main reason for this
is so that your .do scripts don't have to be marked
executable (chmod +x).  Executable .do scripts would
suggest to users that they should run them directly, and
they shouldn't; .do scripts should always be executed
inside an instance of redo, so that dependencies can be
tracked correctly.

WARNING: If your .do script *is* written in Unix sh, we
recommend *not* including the `#!/bin/sh` line.  That's
because there are many variations of /bin/sh, and not all
of them are POSIX compliant.  redo tries pretty hard to
find a good default shell that will be "as POSIXy as
possible," and if you override it using #!/bin/sh, you lose
this benefit and you'll have to worry more about
portability.


# Can a single .do script generate multiple outputs?

FIXME: Yes, but this is a bit imperfect.

For example, compiling a .java file produces a bunch of .class
files, but exactly which files?  It depends on the content
of the .java file.  Ideally, we would like to allow our .do
file to compile the .java file, note which .class files
were generated, and tell redo about it for dependency
checking.

However, this ends up being confusing; if myprog depends
on foo.class, we know that foo.class was generated from
bar.java only *after* bar.java has been compiled.  But how
do you know, the first time someone asks to build myprog,
where foo.class is supposed to come from?

So we haven't thought about this enough yet.

Note that it's *okay* for a .do file to produce targets
other than the advertised one; you just have to be careful. 
You could have a default.javac.do that runs 'javac
$1.java', and then have your program depend on a bunch of .javac
files.  Just be careful not to depend on the .class files
themselves, since redo won't know how to regenerate them.

This feature would also be useful, again, with ./configure:
typically running the configure script produces several
output files, and it would be nice to declare dependencies
on all of them.


# Recursive make is considered harmful.  Isn't redo even *more* recursive?

You probably mean [this 1997 paper](http://miller.emu.id.au/pmiller/books/rmch/)
by Peter Miller.

Yes, redo is recursive, in the sense that every target is built by its own
`.do` file, and every `.do` file is a shell script being run recursively
from other shell scripts, which might call back into `redo`.  In fact, it's
even more recursive than recursive make.  There is no
non-recursive way to use redo.

However, the reason recursive make is considered harmful is that each
instance of make has no access to the dependency information seen by the
other instances.  Each one starts from its own Makefile, which only has a
partial picture of what's going on; moreover, each one has to
stat() a lot of the same files over again, leading to slowness.  That's
the thesis of the "considered harmful" paper.

Nobody has written a paper about it, but *non-recursive*
make should also be considered harmful!  The problem is Makefiles aren't
very "hygienic" or "modular"; if you're not running make recursively, then
your one copy of make has to know *everything* about *everything* in your
entire project.  Every variable in make is global, so every variable defined
in *any* of your Makefiles is visible in *all* of your Makefiles.  Every
little private function or macro is visible everywhere.  In a huge project
made up of multiple projects from multiple vendors, that's just not okay.
Plus, if all your Makefiles are tangled together, make has
to read and parse the entire mess even to build the
smallest, simplest target file, making it slow.

`redo` deftly dodges both the problems of recursive make
and the problems of non-recursive make.  First of all,
dependency information is shared through a global persistent `.redo`
database, which is accessed by all your `redo` instances at once. 
Dependencies created or checked by one instance can be immediately used by
another instance.  And there's locking to prevent two instances from
building the same target at the same time.  So you get all the "global
dependency" knowledge of non-recursive make.  And it's a
binary file, so you can just grab the dependency
information you need right now, rather than going through
everything linearly.

Also, every `.do` script is entirely hygienic and traceable; `redo`
discourages the use of global environment variables, suggesting that you put
settings into files (which can have timestamps and dependencies) instead. 
So you also get all the hygiene and modularity advantages of recursive make.

By the way, you can trace any `redo` build process just by reading the `.do`
scripts from top to bottom.  Makefiles are actually a collection of "rules"
whose order of execution is unclear; any rule might run at any time.  In a
non-recursive Makefile setup with a bunch of included files, you end up with
lots and lots of rules that can all be executed in a random order; tracing
becomes impossible.  Recursive make tries to compensate for this by breaking
the rules into subsections, but that ends up with all the "considered harmful"
paper's complaints.  `redo` runs your scripts from top to bottom in a
nice tree, so it's traceable no matter how many layers you have.


# How do I set environment variables that affect the entire build?

Directly using environment variables is a bad idea because you can't declare
dependencies on them.  Also, if there were a file that contained a set of
variables that all your .do scripts need to run, then `redo` would have to
read that file every time it starts (which is frequently, since it's
recursive), and that could get slow.

Luckily, there's an alternative.  Once you get used to it, this method is
actually much better than environment variables, because it runs faster
*and* it's easier to debug.

For example, djb often uses a computer-generated script called `compile` for
compiling a .c file into a .o file.  To generate the `compile` script, we
create a file called `compile.do`:
	
	redo-ifchange config.sh
	. ./config.sh
	echo "gcc -c -o \$3 $1.c $CFLAGS" >$3
	chmod a+x $3

Then, your `default.o.do` can simply look like this:

	redo-ifchange compile $1.c
	./compile $1 $2 $3

This is not only elegant, it's useful too.  With make, you have to always
output everything it does to stdout/stderr so you can try to figure out
exactly what it was running; because this gets noisy, some people write
Makefiles that deliberately hide the output and print something friendlier,
like "Compiling hello.c".  But then you have to guess what the compile
command looked like.

With redo, the command *is* `./compile hello.c`, which looks good when
printed, but is also completely meaningful.  Because it doesn't depend on
any environment variables, you can just run `./compile hello.c` to reproduce
its output, or you can look inside the `compile` file to see exactly what
command line is being used.

As a bonus, all the variable expansions only need to be done once: when
generating the ./compile program.  With make, it would be recalculating
expansions every time it compiles a file.  Because of the
way make does expansions as macros instead of as normal
variables, this can be slow.


# How do I write a default.o.do that works for both C and C++ source?

We can upgrade the compile.do from the previous answer to
look something like this:

        redo-ifchange config.sh
        . ./config.sh
        cat <<-EOF
                [ -e "\$1.cc" ] && EXT=.cc || EXT=.c
                gcc -o "\$3" -c "\$1\$EXT" -Wall $CFLAGS
        EOF
        chmod a+x "$3"

Isn't it expensive to have ./compile doing this kind of test for every
single source file?  Not really.  Remember, if you have two implicit rules
in make:

	%.o: %.cc
		gcc ...

	%.o: %.c
		gcc ...
		
Then it has to do all the same checks.  Except make has even *more* implicit
rules than that, so it ends up trying and discarding lots of possibilities
before it actually builds your program.  Is there a %.s?  A
%.cpp?  A %.pas?  It needs to look for *all* of them, and
it gets slow.  The more implicit rules you have, the slower
make gets.

In redo, it's not implicit at all; you're specifying exactly how to
decide whether it's a C program or a C++ program, and what to do in each
case.  Plus you can share the two gcc command lines between the two rules,
which is hard in make.  (In GNU make you can use macro functions, but the
syntax for those is ugly.)


# Can I just rebuild a part of the project?

Absolutely!  Although `redo` runs "top down" in the sense of one .do file
calling into all its dependencies, you can start at any point in the
dependency tree that you want.

Unlike recursive make, no matter which subdir of your project you're in when
you start, `redo` will be able to build all the dependencies in the right
order.

Unlike non-recursive make, you don't have to jump through any strange hoops
(like adding, in each directory, a fake Makefile that does `make -C ${TOPDIR}`
back up to the main non-recursive Makefile).  redo just uses `filename.do`
to build `filename`, or uses `default*.do` if the specific `filename.do`
doesn't exist.

When running any .do file, `redo` makes sure its current directory is set to
the directory where the .do file is located.  That means you can do this:

	redo ../utils/foo.o
	
And it will work exactly like this:

	cd ../utils
	redo foo.o
	
In make, if you run

	make ../utils/foo.o
	
it means to look in ./Makefile for a rule called
../utils/foo.o... and it probably doesn't have such a
rule.  On the other hand, if you run

	cd ../utils
	make foo.o
	
it means to look in ../utils/Makefile and look for a rule
called foo.o.  And that might do something totally
different!  redo combines these two forms and does
the right thing in both cases.

Note: redo will always change to the directory containing
the target before trying to build it.  So if you do

	redo ../utils/foo.o

the .do file will be run with its current directory set to
../utils.  Thus, the .do file's runtime environment is
always reliable.


# Can my filenames have spaces in them?

Yes, unlike with make.  For historical reasons, the Makefile syntax doesn't
support filenames with spaces; spaces are used to separate one filename from
the next, and there's no way to escape these spaces.

Since redo just uses sh, which has working escape characters and
quoting, it doesn't have this problem.


# Does redo care about the differences between tabs and spaces?

No.


# What if my .c file depends on a generated .h file?

This problem arises as follows.  foo.c includes config.h, and config.h is
created by running ./configure.  The second part is easy; just write a
config.h.do that depends on the existence of configure (which is created by
configure.do, which probably runs autoconf).

The first part, however, is not so easy.  Normally, the headers that a C
file depends on are detected as part of the compilation process.  That works
fine if the headers, themselves, don't need to be generated first.  But if
you do

	redo foo.o
	
There's no way for redo to *automatically* know that compiling foo.c
into foo.o depends on first generating config.h.

Since most .h files are *not* auto-generated, the easiest
thing to do is probably to just add a line like this to
your default.o.do:

	redo-ifchange config.h
	
Sometimes a specific solution is much easier than a general
one.

If you really want to solve the general case,
[djb has a solution for his own
projects](http://cr.yp.to/redo/honest-nonfile.html), which is a simple
script that looks through C files to pull out #include lines.  He assumes
that `#include <file.h>` is a system header (thus not subject to being
built) and `#include "file.h"` is in the current directory (thus easy to
find).  Unfortunately this isn't really a complete
solution, but at least it would be able to redo-ifchange a
required header before compiling a program that requires
that header.


# Why doesn't redo by default print the commands as they are run?

make prints the commands it runs as it runs them.  redo doesn't, although
you can get this behaviour with `redo -v` or `redo -x`. 
(The difference between -v and -x is the same as it is in
sh... because we simply forward those options onward to sh
as it runs your .do script.)

The main reason we don't do this by default is that the commands get
pretty long
winded (a compiler command line might be multiple lines of repeated
gibberish) and, on large projects, it's hard to actually see the progress of
the overall build.  Thus, make users often work hard to have make hide the
command output in order to make the log "more readable."

The reduced output is a pain with make, however, because if there's ever a
problem, you're left wondering exactly what commands were run at what time,
and you often have to go editing the Makefile in order to figure it out.

With redo, it's much less of a problem.  By default, redo produces output
that looks like this:

	$ redo t
	redo  t/all
	redo    t/hello
	redo      t/LD
	redo      t/hello.o
	redo        t/CC
	redo    t/yellow
	redo      t/yellow.o
	redo    t/bellow
	redo    t/c
	redo      t/c.c
	redo        t/c.c.c
	redo          t/c.c.c.b
	redo            t/c.c.c.b.b
	redo    t/d

The indentation indicates the level of recursion (deeper levels are
dependencies of earlier levels).  The repeated word "redo" down the left
column looks strange, but it's there for a reason, and the reason is this:
you can cut-and-paste a line from the build script and rerun it directly.

	$ redo t/c
	redo  t/c
	redo    t/c.c
	redo      t/c.c.c
	redo        t/c.c.c.b
	redo          t/c.c.c.b.b

So if you ever want to debug what happened at a particular step, you can
choose to run only that step in verbose mode:

	$ redo t/c.c.c.b.b -x
	redo  t/c.c.c.b.b
	* sh -ex default.b.do c.c.c.b .b c.c.c.b.b.redo2.tmp
	+ redo-ifchange c.c.c.b.b.a
	+ echo a-to-b
	+ cat c.c.c.b.b.a
	+ ./sleep 1.1
	redo  t/c.c.c.b.b (done)
	

If you're using an autobuilder or something that logs build results for
future examination, you should probably set it to always run redo with
the -x option.


# Is redo compatible with autoconf?

Yes.  You don't have to do anything special, other than the above note about
declaring dependencies on config.h, which is no worse than what you would
have to do with make.


# Is redo compatible with automake?

Hells no.  You can thank me later.  But see next question.


# Is redo compatible with make?

Yes.  If you have an existing Makefile (for example, in one of your
subprojects), you can just call make from a .do script to build that
subproject.

In a file called myproject.stamp.do:

	redo-ifchange $(find myproject -name '*.[ch]')
	make -C myproject all

So, to amend our answer to the previous question, you *can* use
automake-generated Makefiles as part of your redo-based project.


# Is redo -j compatible with make -j?

Yes!  redo implements the same jobserver protocol as GNU make, which means
that redo running under make -j, or make running under redo -j, will do the
right thing.  Thus, it's safe to mix-and-match redo and make in a recursive
build system.

Just make sure you declare your dependencies correctly;
redo won't know all the specific dependencies included in
your Makefile, and make won't know your redo dependencies,
of course.

One way of cheating is to just have your make.do script
depend on *all* the source files of a subproject, like
this:

	make -C subproject all
	find subproject -name '*.[ch]' | xargs redo-ifchange

Now if any of the .c or .h files in subproject are changed,
your make.do will run, which calls into the subproject to
rebuild anything that might be needed.  Worst case, if the
dependencies are too generous, we end up calling 'make all'
more often than necessary.  But 'make all' probably runs
pretty fast when there's nothing to do, so that's not so
bad.


# Parallelism if more than one target depends on the same subdir

Recursive make is especially painful when it comes to
parallelism.  Take a look at this Makefile fragment:

	all: fred bob
	subproj:
		touch $@.new
		sleep 1
		mv $@.new $@
	fred:
		$(MAKE) subproj
		touch $@
	bob:
		$(MAKE) subproj
		touch $@

If we run it serially, it all looks good:

	$ rm -f subproj fred bob; make --no-print-directory
	make subproj
	touch subproj.new
	sleep 1
	mv subproj.new subproj
	touch fred
	make subproj
	make[1]: 'subproj' is up to date.
	touch bob
	
But if we run it in parallel, life sucks:

	$ rm -f subproj fred bob; make -j2 --no-print-directory
	make subproj
	make subproj
	touch subproj.new
	touch subproj.new
	sleep 1
	sleep 1
	mv subproj.new subproj
	mv subproj.new subproj
	mv: cannot stat 'ubproj.new': No such file or directory
	touch fred
	make[1]: *** [subproj] Error 1
	make: *** [bob] Error 2
	
What happened?  The sub-make that runs `subproj` ended up
getting twice at once, because both fred and bob need to
build it.

If fred and bob had put in a *dependency* on subproj, then
GNU make would be smart enough to only build one of them at
a time; it can do ordering inside a single make process. 
So this example is a bit contrived.  But imagine that fred
and bob are two separate applications being built from the
same toplevel Makefile, and they both depend on the library
in subproj.  You'd run into this problem if you use
recursive make.

Of course, you might try to solve this by using
*nonrecursive* make, but that's really hard.  What if
subproj is a library from some other vendor?  Will you
modify all their makefiles to fit into your nonrecursive
makefile scheme?  Probably not.

Another common workaround is to have the toplevel Makefile
build subproj, then fred and bob.  This works, but if you
don't run the toplevel Makefile and want to go straight
to work in the fred project, building fred won't actually
build subproj first, and you'll get errors.

redo solves all these problems.  It maintains global locks
across all its instances, so you're guaranteed that no two
instances will try to build subproj at the same time.  And
this works even if subproj is a make-based project; you
just need a simple subproj.do that runs `make subproj`.


# Dependency problems that only show up during parallel builds

One annoying thing about parallel builds is... they do more
things in parallel.  A very common problem in make is to
have a Makefile rule that looks like this:

	all: a b c
	
When you `make all`, it first builds a, then b, then c. 
What if c depends on b?  Well, it doesn't matter when
you're building in serial.  But with -j3, you end up
building a, b, and c at the same time, and the build for c
crashes.  You *should* have said:

	all: a b c
	c: b
	b: a
	
and that would have fixed it.  But you forgot, and you
don't find out until you build with exactly the wrong -j
option.

This mistake is easy to make in redo too.  But it does have
a tool that helps you debug it: the --shuffle option. 
--shuffle takes the dependencies of each target, and builds
them in a random order.  So you can get parallel-like
results without actually building in parallel.


# What about distributed builds?

FIXME:
So far, nobody has tried redo in a distributed build environment.  It surely
works with distcc, since that's just a distributed compiler.  But there are
other systems that distribute more of the build process to other machines.

The most interesting method I've heard of was explained (in public, this is
not proprietary information) by someone from Google.  Apparently, the
Android team uses a tool that mounts your entire local filesystem on a
remote machine using FUSE and chroots into that directory.  Then you replace
the $SHELL variable in your copy of make with one that runs this tool. 
Because the remote filesystem is identical to yours, the build will
certainly complete successfully.  After the $SHELL program exits, the changed
files are sent back to your local machine.  Cleverly, the files on the
remote server are cached based on their checksums, so files only need to be
re-sent if they have changed since last time.  This dramatically reduces
bandwidth usage compared to, say, distcc (which mostly just re-sends the
same preparsed headers over and over again).

At the time, he promised to open source this tool eventually.  It would be
pretty fun to play with it.

The problem:

This idea won't work as easily with redo as it did with
make.  With make, a separate copy of $SHELL is launched for
each step of the build (and gets migrated to the remote
machine), but make runs only on your local machine, so it
can control parallelism and avoid building the same target
from multiple machines, and so on.  The key to the above
distribution mechanism is it can send files to the remote
machine at the beginning of the $SHELL, and send them back
when the $SHELL exits, and know that nobody cares about
them in the meantime.  With redo, since the entire script
runs inside a shell (and the shell might not exit until the
very end of the build), we'd have to do the parallelism
some other way.

I'm sure it's doable, however.  One nice thing about redo
is that the source code is so small compared to make: you
can just rewrite it.


# Can I convince a sub-redo or sub-make to *not* use parallel builds?

Yes.  Put this in your .do script:

	unset MAKEFLAGS
	
The child makes will then not have access to the jobserver,
so will build serially instead.


# How fast is redo compared to make?

FIXME:
The current version of redo is written in python and has not been optimized. 
So right now, it's usually a bit slower.  Not too embarrassingly slower,
though, and the slowness mostly only strikes when you're
building a project from scratch.

For incrementally building only the changed parts of the project, redo can
be much faster than make, because it can check all the dependencies up
front and doesn't need to repeatedly parse and re-parse the Makefile (as
recursive make needs to do).

redo's sqlite3-based dependency database is very fast (and
it would be even faster if we rewrite redo in C instead of
python).  Better still, it would be possible to write an
inotify daemon that can update the dependency database in
real time; if you're running the daemon, you can run 'redo'
from the toplevel and if your build is clean, it could return
instantly, no matter how many dependencies you have.

On my machine, redo can currently check about 10,000
dependencies per second.  As an example, a program that
depends on every single .c or .h file in the Linux kernel
2.6.36 repo (about 36000 files) can be checked in about 4
seconds.

Rewritten in C, dependency checking would probably go about
10 times faster still.

This probably isn't too hard; the design of redo is so simple that
it should be easy to write in any language.  It's just
*even easier* in python, which was good for writing the
prototype and debugging the parallelism and locking rules.

Most of the slowness at the moment is because redo-ifchange
(and also sh itself) need to be fork'd and exec'd over and
over during the build process.

As a point of reference, on my computer, I can fork-exec
redo-ifchange.py about 87 times per second; an empty python
program, about 100 times per second; an empty C program,
about 1000 times per second; an empty make, about 300 times
per second.  So if I could compile 87 files per second with
gcc, which I can't because gcc is slower than that, then
python overhead would be 50%.  Since gcc is slower than
that, python overhead is generally much less - more like
10%.

Also, if you're using redo -j on a multicore machine, all
the python forking happens in parallel with everything
else, so that's 87 per second per core.  Nevertheless,
that's still slower than make and should be fixed.

(On the other hand, all this measurement is confounded
because redo's more fine-grained dependencies mean you can
have more parallelism.  So if you have a lot of CPU cores, redo
might build *faster* than make just because it makes better
use of them.)


# The output of 'ps ax' is ugly because of the python interpreter!

FIXME:
Yes, this is a general problem with python.  All the lines
in 'ps' end up looking like

	28460 pts/2 Sl 0:00 /usr/bin/python /path/to/redo-ifchange stuff...

...which means that the "stuff..." part is often cut off on
the right hand side.  There are gross workarounds to fix
this in python, but the easiest fix will be to just rewrite
redo in C.  Then it'll look like this:

	28460 pts/2 Sl 0:00 redo-ifchange stuff...

...the way it should.
	

# Are there examples?

FIXME: There are some limited ones in the `t/example/` subdir of
the redo project.  The best example is a real, live program
using redo as a build process.  If you switch your
program's build process to use redo, please let us know and
we can link to it here.


# What's missing?  How can I help?

redo is incomplete and probably has numerous bugs.  Just what you
always wanted in a build system, I know.

What's missing?  Search for the word FIXME in this document; anything with a
FIXME is something that is either not implemented, or which needs
discussion, feedback, or ideas.  Of course, there are surely other
undocumented things that need discussion or fixes too.

You should join the redo-list@googlegroups.com mailing list.

You can find the mailing list archives here:

	http://groups.google.com/group/redo-list

Yes, it might not look like it, but you can subscribe without having a
Google Account.  Just send a message here:

	redo-list+subscribe@googlegroups.com

Note the plus sign.

Have fun,

Avery
