---
layout: post
title: "Build Snippets #1 - Affected Target Analysis with Bazel"
description: "Clever tricks with bazel-diff"
date: 2025-09-28
tags: [bazel, snippets]
---

# Affected target analysis

A question that naturally arises when interacting with pull requests is "What does this change affect?".
It's pretty easy to get Git to tell us[^git] what files were altered between two commits but if you answered that question with the list of modified files your interlocutor would probably roll their eyes at you.
Clearly, not all files are equal in significance: A change to a `README` probably merits less scrutiny & testing than a change to a unit test and changes to either of those are less significant (in some sense) than a change to a core behavioral library or user-facing API. 
Sometimes the connection between a critical component and a file-level diff can be much harder to understand: An edit to a build script, or a change to toolchain, or rolling the URI pointing to a pinned dependency could all cause changes of great significance in ways that are less obvious than an edit to `important_library.cc`.

<!--more-->

Fortunately, Bazel both has a holistic view of all of these relationships in a project and provides nice query mechanisms for us to be able to answer our original question.
The bulk of this analysis is powered by the excellent [bazel-diff](https://github.com/Tinder/bazel-diff) tool.

`bazel-diff` is quite easy to set up & use[^usage], so we'll assume we've already run it against two commits and have written the list of affected targets to `affected_targets.txt`.
The contents of that file are a list of canonicalized label names, one per line, looking something like:
```
//foo:bar
//foo:foo
//foo/baz:qux
//spam/ham:breakfast
<typically many more lines...>
``` 

In case you are not already familiar with `bazel-diff` the property of "being affected" propagates through dependencies:
If target A is affected and target B depends on target A then target B will also be considered to be affected, and so on transitively[^deps].

Great, now we know everything that Bazel thinks has been potentially affected and we've answered the original question.
But what do we *do* with that information? 
It turns out the question we're usually actually interested in isn't "What has been affected?" (a set of targets) but instead "Has anything important been affected?" (a simple yes/no).

To answer that we need to define what it means to be important!
I like to think of this in terms of particular jobs or tasks that then get to declare what is important *to that particular job*.
Maybe you want to run some release process if the server binary has been affected or selectively start up an expensive test job that uses a Bazel-produced binary but only if that binary has potentially changed.
Since this entire process of affected target analysis is typically happening in the context of a CI job, we'll define the targets we consider important in the *de facto* language of CI: Bash.

```bash
IMPORTANT_TARGETS=(
    //services/galactus:server
    //services/galactus:config
)
```

In the above snippet our array of important targets consists of Bazel labels which matches the shape of the information we have in `affected_targets.txt`.
There are easy-to-imagine situations where the targets we consider important fit a more general category like "any target in this directory".
It would be a real drag for us (or our users) to have to type those all out by hand and maintain the list over time so we'd like to be able to use [target patterns](https://bazel.build/versions/8.4.0/run/build#specifying-build-targets) in our list of important targets:

```bash
IMPORTANT_TARGETS=(
    //services/galactus:server
    //services/galactus:config
    //deploy/galactus/...
)
```

We can easily support this by using `bazel query` to expand & canonicalize the list of labels[^list] or patterns into a full list of labels in a file named `important_targets.txt`:

Code snippet:
```bash
bazel query 'set('"${RELEVANT_TARGETS[*]}"')' --output_file=important_targets.txt
```

It turns out that `bazel query` is pretty permissive in what it accepts as names for targets. In addition to target labels you can pass in relative file paths to any file that is a dependency of a declared target or to bare files mentioned in an `exports_files` directive. So our target list could even look like:

```bash
IMPORTANT_TARGETS=(
    //services/galactus:server
    //services/galactus:config
    //deploy/galactus/...
    omegastar/docs/MANIFESTO.md
)
```

Now answering the question "Has anything important been affected?" is quite straightforward: Is any entry in `important_targets.txt` present in `affected_targets.txt`?
The principled way to do this would be to ingest each file into a set-like structure and check to see if the intersection is nonempty, but let's be Build gremlins together and do it in a Bash one-liner:
```bash
if grep -q -x -f important_targets.txt affected_targets.txt ;
    run_expensive_job
fi
```

This is taking advatange of the fact `grep` exits with 0 if a match was found and 1 if no match was found, along with the following option flags:

* `-q`/`--quiet`: Suppresses printing the matches, since we don't actually care *what* is matched only whether or not a match is found.
* `-f`/`--file=`: Obtain the search patterns from the specified file, one per line.
* `-x`/`--line-regexp`: Treat search patterns as only matching whole lines, i.e. implicitly surround the search pattern with `^` and `$`. This is needed so that labels only match themselves and not other labels that they are proper prefixes of, e.g. `//foo/bar:baz` should not match `//foo/bar:baz_v2`.

With that we have all the necessary pieces to skip or dynamically trigger jobs based on Bazel's analysis of what targets have been affected. Stop running those expensive jobs on `README` typo fixes!

## Addendum: But Why Did This Run?

A piece of feedback I quickly heard on this system was that it was sometimes hard to understand *why* a particular job had considered its important targets to be affected.
Programatically answering this in full generality is challenging[^changes] but I came up with a simple query that works as a solid heuristic for providing a useful answer to a curious human:

```python
somepath(
    set(IMPORTANT_TARGETS),
    kind("source file", set(AFFECTED_TARGETS))
)
```

Simply put: "Find some chain of dependencies from one of the important targets to a source file that was changed."

Often times the list of affected targets is **very large** and so we typically can't construct this query string as a command line argument. `bazel query` takes a `--query_file` argument to work around this and we can build that query file with a bit more Bash. Starlark isn't sensitive to excess whitespace so we can generate the query very succinctly like:
```bash
{
    echo "somepath(set("
    cat important_targets.txt
    echo '), kind("source file", set('
    cat affected_targets.txt
    echo ")))"
} > query_file.txt
```

All of the `echo` and `cat` commands are run in a [command group](https://www.gnu.org/software/bash/manual/bash.html#Grouping-Commands) and we redirect the output of the entire group to a file.

---

[^git]: `git diff --name-only <commit1> <commit2>`, for reference.

[^usage]: You can locally clone the `bazel-diff` repo and run [bazel-diff-example.sh](https://github.com/Tinder/bazel-diff/blob/master/bazel-diff-example.sh) with the path to your project as the first argument, or you can import the whole `bazel-diff` repo as an external repository in your project and write your own wrapper script.

[^deps]: In actuality the relationship is a more subtle & complicated than this since Bazel targets are really an abstraction layer that can correspond to configuration pieces or actions.
It would be more accurate to say that target B will be considered affected if one of its actions consumes a file produced by one of target A's actions.
A counterexample to the simple explanation I gave of transitive propagation would be a dependency in the [implementation_deps](https://bazel.build/versions/8.4.0/reference/be/c-cpp#cc_library.implementation_deps) attribute of a `cc_library` rule:
If we have `cc_library` targets `//:foo`, `//:bar`, and `//:baz` with `foo` an `implementation_dep` of `bar` and `bar` a regular dependency of `baz`, then if `foo.h` in `//:foo` is changed `//:foo` and `//:bar` will be considered affected but `//:baz` will not be as the change to `foo.h` does not affect any of the files belonging to `//:bar` that `//:baz` actually consumes, i.e. `bar`'s header files.

[^list]: If your list of targets is so long that expanding it results in a query string that is too big to pass as an argument you can work around that via `--query_file` and some Bash contortions: `bazel query --query_file=<(echo 'set('; printf '%s\n' "${RELEVANT_TARGETS[@]}" ; echo ')' )`.
`printf` is a shell builtin rather than an external command and so avoids the kernel-imposed length limits on command-line arguments.

[^changes]: Changing the version of Bazel itself or changing certain command-line flags in `bazelrc` will result in the analysis showing that literally every target was affected with no useful causal chain.