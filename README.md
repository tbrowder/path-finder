[![Actions Status](https://github.com/tbrowder/path-finder/actions/workflows/linux.yml/badge.svg)](https://github.com/tbrowder/path-finder/actions) [![Actions Status](https://github.com/tbrowder/path-finder/actions/workflows/macos.yml/badge.svg)](https://github.com/tbrowder/path-finder/actions) [![Actions Status](https://github.com/tbrowder/path-finder/actions/workflows/windows-spec.yml/badge.svg)](https://github.com/tbrowder/path-finder/actions)

SYNOPSIS
========

    use Path::Finder;

    my $finder = Path::Finder.file.skip-vcs.ext(/pm6?/).size(* > 10_000);

    # iterator interface
    for $finder.in('.') -> $file {
      ...
    }

    # functional interface
    for find(:file, :skip-vcs, :ext(/pm6?/), :size(* > 10_000)) -> $file {
      ...
    }

DESCRIPTION
===========

This module iterates over files and directories to identify ones matching a user-defined set of rules. The object-oriented API is based heavily on perl5's `Path::Iterator::Rule`. A `Path::Finder` object is a collection of rules (match criteria) with methods to add additional criteria. Options that control directory traversal are given as arguments to the method that generates an iterator.

Here is a summary of features for comparison to other file finding modules:

  * provides many "helper" methods for specifying rules

  * offers (lazy) list interface

  * custom rules implemented with roles

  * breadth-first (default) or pre- or post-order depth-first searching

  * follows symlinks (by default, but can be disabled)

  * directories visited only once (no infinite loop; can be disabled)

  * doesn't chdir during operation

  * provides an API for extensions

USAGE
=====

There are two interfaces: an object oriented one, and a functional one.

Path::Finder objects are immutable. All methods except `in` return a new object combining the existing rules with the additional rules provided.

When using the `find` function, all methods described below (except `in`) are allowed as named arguments, as well as all arguments to `in`. This is usually the easiest way to use Path::Finder, even if it allows for slightly less control over ordering of the constraints. There is also a `finder` function that returns a Path::Finder object.

Matching and iteration
----------------------

### `CALL-ME` / `in`

    for $finder(@dirs, |%options) -> $file {
      ...
    }

Creates a sequence of results. This sequence is "lazy" -- results are not pre-computed.

It takes as arguments a list of directories to search and named arguments as control options. Valid options include:

  * `order` -- Controls order of results. Valid values are `BreadthFirst` (breadth-first search), `PreOrder` (pre-order, depth-first search), `PostOrder` (post-order, depth-first search). The default is `PreOrder`.

  * `follow-symlinks` - Follow directory symlinks when true. Default is `True`.

  * `report-symlinks` - Includes symlinks in results when true. Default is equal to `follow-symlinks`.

  * `loop-safe` - Prevents visiting the same directory more than once when true. Default is `True`.

  * `relative` - Return matching items relative to the search directory. Default is `False`.

  * `sorted` - Whether entries in a directory are sorted before processing. Default is `True`.

  * `keep-going` - Whether or not the search should continue when an error is encountered (typically an unreadable directory). Defaults to `True`.

  * `quiet` - Whether printing non-fatal errors to `$*ERR` is repressed. Defaults to `False`.

  * `invert` - This will invert which files are matched and which files are not

  * `as` - The type of values that will be returned. Valid values are `IO::Path` (the default) and `Str`.

Filesystem loops might exist from either hard or soft links. The `loop-safe` option prevents infinite loops, but adds some overhead by making `stat` calls. Because directories are visited only once when `loop-safe` is true, matches could come from a symlinked directory before the real directory depending on the search order.

To get only the real files, turn off `follow-symlinks`. You can have symlinks included in results, but not descend into symlink directories if you turn off `follow-symlinks`, but turn on `report-symlinks`.

Turning `loop-safe` off and leaving `follow-symlinks` on avoids `stat` calls and will be fastest, but with the risk of an infinite loop and repeated files. The default is slow, but safe.

If the search directories are absolute and the `relative` option is true, files returned will be relative to the search directory. Note that if the search directories are not mutually exclusive (whether containing subdirectories like `@*INC` or symbolic links), files found could be returned relative to different initial search directories based on `order`, `follow-symlinks` or `loop-safe`.

Logic operations
----------------

`Path::Finder` provides three logic operations for adding rules to the object. Rules may be either a subroutine reference with specific semantics or another `Path::Finder` object.

### `and`

    $finder.and($finder2) ;
    $finder.and(-> $item, *%) { $item ~~ :rwx });
    $finder.and(@more-rules);

    find(:and(@more-rules));

This creates a new rule combining the curent one and the arguments. E.g. "old rule AND new1 AND new2 AND ...".

### `or`

    $finder.or(
      Path::Finder.name("foo*"),
      Path::Finder.name("bar*"),
      -> $item, *% { $item ~~ :rwx },
    );

This creates a new rule combining the curent one and the arguments. E.g. "old rule OR new1 OR new2 OR ...".

### `none`

    $finder.none( -> $item, *% { $item ~~ :rwx } );

This creates a new rule combining the current one and one or more alternatives and adds them as a negative constraint to the current rule. E.g. "old rule AND NOT ( new1 AND new2 AND ...)". Returns the object to allow method chaining.

### `not`

    $finder.not();

This creates a new rule negating the whole original rule. Returns the object to allow method chaining.

### `skip`

    $finder.skip(
      $finder.new.dir.not-writeable,
      $finder.new.dir.name("foo"),
    );

Takes one or more alternatives and will prune a directory if any of the criteria match or if any of the rules already indicate the directory should be pruned. Pruning means the directory will not be returned by the iterator and will not be searched.

For files, it is equivalent to `$finder.none(@rules) `. Returns the object to allow method chaining.

This method should be called as early as possible in the rule chain. See `skip-dir` below for further explanation and an example.

RULE METHODS
============

Rule methods are helpers that add constraints. Internally, they generate a closure to accomplish the desired logic and add it to the rule object with the `and` method. Rule methods return the object to allow for method chaining.

Generally speaking there are two kinds of rule methods: the ones that take a value to smartmatch some property against (e.g. `name`), and ones that take a boolean (defaulting to `True`) to check a boolean value against (e.g. `readable`).

File name rules
---------------

### `name`

    $finder.name("foo.txt");
    find(:name<foo.txt>);

The `name` method takes a pattern and creates a rule that is true if it matches the **basename** of the file or directory path. Patterns may be anything that can smartmatch a string. If it's a string it will be interpreted as a glob pattern.

### `path`

    $finder.path( "foo/*.txt" );
    find(:path<foo/*.txt>);

The `path` method takes a pattern and creates a rule that is true if it matches the path of the file or directory. Patterns may be anything that can smartmatch a string. If it's a string it will be interpreted as a glob pattern.

### `relpath`

    $finder.relpath( "foo/bar.txt" );
    find(:relpath<foo/bar.txt>);

    $finder.relpath( any(rx/foo/, "bar.*"));
    find(:relpath(any(rx/foo/, "bar.*"))

The `relpath` method takes a pattern and creates a rule that is true if it matches the path of the file or directory relative to its basedir. Patterns may be anything that can smartmatch a string. If it's a string it will be interpreted as a glob pattern.

### `io`

    $finder.path(:f|:d);
    find(:path(:f|:d);

The `io` method takes a pattern and creates a rule that is true if it matches the `IO` of the file or directory. This is mainly useful for combining filetype tests.

### `ext`

The `name` method takes a pattern and creates a rule that is true if it matches the extension of path. Patterns may be anything that can smartmatch a string.

### `skip-dir`

    $finder.skip-dir( $pattern );

The `skip-dir` method skips directories that match a pattern. Directories that match will not be returned from the iterator and will be excluded from further search. **This includes the starting directories.** If that isn't what you want, see `skip-subdir` instead.

**Note:** this rule should be specified early so that it has a chance to operate before a logical shortcut. E.g.

    $finder.skip-dir(".git").file; # OK
    $finder.file.skip-dir(".git"); # Won't work

In the latter case, when a ".git" directory is seen, the `file` rule shortcuts the rule before the `skip-dir` rule has a chance to act.

### `skip-subdir`

    $finder.skip-subdir( @patterns );

This works just like `skip-dir`, except that the starting directories (depth 0) are not skipped and may be returned from the iterator unless excluded by other rules.

File test rules
---------------

Most of the `:X` style filetest are available as boolean rules:

### `readable`

This checks if the entry is readable

### `writable`

This checks if the entry is writable

### `executable`

This checks if the entry is executable

### `file`

This checks if the entry is a file

### `directory`

This checks if the entry is a directory

### `symlink`

This checks if the entry is a symlink

### `special`

This checks if the entry is anything but a file, directory or symlink.

### `exists`

This checks if the entry exists

### `empty`

This checks if the entry is empty

For example:

    $finder.file.empty;

Two composites are also available:

### `read-writable`

This checks if the entry is readable and writable

### `read-write-executable`

This checks if the entry is readable, writable and executable

### `dangling`

    $finder.dangling;

The `dangling` rule method matches dangling symlinks. It's equivalent to

    $finder.symlink.exists(False)

The timestamps methods take a single argument in a form that can smartmatch an `Instant`.

### `accessed`

Compares the access time

### `modified`

Compares the modification time

### `changed`

Compares the (inode) change time

For example:

    # hour old
    $finder.modified(* < now - 60 * 60);

It also supports the following integer based matching rules:

### `size`

This compares the size of the entry

### `mode`

This compares the mode of the entry

### `device`

This compares the device of the entry. This may not be available everywhere.

### `device-type`

This compares the device ID of the entry (when its a special file). This may not be available everywhere.

### `inode`

This compares the inode of the entry. This may not be available everywhere.

### `nlinks`

This compares the link count of the entry. This may not be available everywhere.

### `uid`

This compares the user identifier of the entry.

### `gid`

This compares the group identifier of the entry.

### `blocks`

This compares the number of blocks in the entry.

### `blocksize`

This compares the blocksize of the entry.

For example:

    $finder.size(* > 10240)

Depth rules
-----------

    $finder.depth(3..5);

The `depth` rule method take a single range argument and limits the paths returned to a minimum or maximum depth (respectively) from the starting search directory, or an integer representing a specific depth. A depth of 0 means the starting directory itself. A depth of 1 means its children. (This is similar to the Unix `find` utility.)

Version control file rules
--------------------------

    # Skip all known VCS files
    $finder.skip-vcs;

Skips files and/or prunes directories related to a version control system. Just like `skip-dir`, these rules should be specified early to get the correct behavior.

File content rules
------------------

### `contents`

    $finder.contents(rx/BEGIN .* END/);

The `contents` rule takes a list of regular expressions and returns files that match one of the expressions.

The expressions are applied to the file's contents as a single string. For large files, this is likely to take significant time and memory.

Files are assumed to be encoded in UTF-8, but alternative encodings can be passed as a named argument:

    $finder.contents(rx/BEGIN .* END/xs, :enc<latin1>);

### `lines`

    $finder.lines(rx:i/^new/);

The `line` rule takes a list of regular expressions and returns files with at least one line that matches one of the expressions.

Files are assumed to be encoded in UTF-8, but alternative Perl IO layers can be passed like in `contents`

### `no-lines`

    $finder.no-lines(rx:i/^new/);

The `line` rule takes a list of regular expressions and returns files with no lines that matches one of the expressions.

Files are assumed to be encoded in UTF-8, but alternative Perl IO layers can be passed like in `contents`

### `shebang`

    $finder.shebang(rx/#!.*\bperl\b/);

The `shebang` rule takes a value and checks it against the first line of a file. The default checks for `rx/^#!/`.

Other rules
-----------

EXTENDING
=========

Custom rule subroutines
-----------------------

Rules are implemented as (usually anonymous) subroutine callbacks that return a value indicating whether or not the rule matches. These callbacks are called with three arguments. The only positional argument is a path.

    $finder.and( sub ($item, *%args) { $item ~~ :r & :w & :x } );

The named arguments contain more information for such a check For example, the `depth` key is used to support minimum and maximum depth checks.

The custom rule subroutine must return one of four values:

  * `True` -- indicates the constraint is satisfied

  * `False` -- indicates the constraint is not satisfied

  * `PruneExclusive` -- indicate the constraint is satisfied, and prune if it's a directory

  * `PruneInclusive` -- indicate the constraint is not satisfied, and prune if it's a directory

Here is an example. This is equivalent to the "depth" rule method with a depth of `0..3`:

    $finder.and(
      sub ($path, :$depth, *%) {
        return $depth < 3 ?? True !! PruneExclusive;
      }
    );

Files and directories and directories up to depth 3 will be returned and directories will be searched. Files of depth 3 will be returned. Directories of depth 3 will be returned, but their contents will not be added to the search.

Once a directory is flagged to be pruned, it will be pruned regardless of subsequent rules.

    $finder.depth(0..3).name(rx/foo/);

This will return files or directories with "foo" in the name, but all directories at depth 3 will be pruned, regardless of whether they match the name rule.

Generally, if you want to do directory pruning, you are encouraged to use the `skip` method instead of writing your own logic using `PruneExclusive` and `PruneInclusive`.

PERFORMANCE
===========

By default, `Path::Finder` iterator options are "slow but safe". They ensure uniqueness, return files in sorted order, and throw nice error messages if something goes wrong.

If you want speed over safety, set these options:

    :!loop-safe, :!sorted, :order(PreOrder)

Depending on the file structure being searched, `:order(PreOrder) ` may or may not be a good choice. If you have lots of nested directories and all the files at the bottom, a depth first search might do less work or use less memory, particularly if the search will be halted early (e.g. finding the first N matches.)

Rules will shortcut on failure, so be sure to put rules likely to fail early in a rule chain.

Consider:

    $f1 = Path::Finder.new.name(rx/foo/).file;
    $f2 = Path::Finder.new.file.name(rx/foo/);

If there are lots of files, but only a few containing "foo", then `$f1` above will be faster.

Rules are implemented as code references, so long chains have some overhead. Consider testing with a custom coderef that combines several tests into one.

Consider:

    $f3 = Path::Finder.new.read-write-executable;
    $f4 = Path::Finder.new.readable.writeable.executable;

Rule `$f3` above will be much faster, not only because it stacks the file tests, but because it requires to only check a single rule.

When using the `find` function, `Path::Finder` will try to sort the arguments automatically in such a way that cheap checks and skipping checks are done first.

