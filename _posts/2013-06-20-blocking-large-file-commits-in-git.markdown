---
layout: post
title:  "Blocking large file commits in git"
date:   2013-06-20 17:51:24 -0700 
categories: git
---

In my previous blog post on [Locating large objects in a git
repository][llo], I covered some of the git utilities that can be used to
identify large files in your repository. In this post I'd like to go over how
to create a server-side hook that will block large files before they get into
your repo. Some of the same utilities from the post referenced above will be
used in creating this hook.

# What are server-side hooks?
[Git hooks][githooks] allow you to perform actions at different points in the
git push workflow. Server-side hooks are executed at certain specific phases of
the push process:

* `pre-receive`

  Fires before any refs are updated on the server, and is run once.  This hook
  receives a list of changes on stdin in the format of: `old-ref new-ref
  refname`, one line for each ref to be updated. Both stderr and stdout are
  forwarded back to the user for messaging. A non-zero exit will prevent any
  refs from being updated.

* `update`

  Fires before the updating the refs on the server, and is executed once for
  each ref that is being updated. It takes 3 arguments: `ref-name old-ref
  new-ref`, and will pass stderr and stdout back to the client. A non-zero exit
  status will only prevent the effected ref from being updated on the server.

* `post-receive`
  
  Fires once after all refs have been updated, and get's the same input on
  stdin as the `pre-receive` hook. As with the hooks before it, stdout and
  stderr are passed back to the client for messaging. Generally this type of
  hook is only used for notification and/or messaging to clients about the push
  that just occured.

* `post-update`
  
  As with the `post-receive` hook, this fires once after the refs have been
  updated, and takes a variable number of arguments. Each parameter that is
  passed in, is the name of a ref that was successfully updated. Similarly to
  `post-receive` this hook is also used primarily for notification, as it cannot
  effect the status of the push.

On the git server these hooks live inside the `hooks` directory of the bare
repository. They must to be named to match the type of hook they are.

# Which hook type should I use?

Since we're looking to block an actual commit from happening, we would want to
use either a `pre-receive` or `update` hook. The distinction here is that the
`pre-receive` will block **all updates** on a non-zero exit, whereas the
`update` will only block the specific update for the ref which returns a
non-zero exit. For this example I will choose to use `pre-receive` because I
want to block the entire push if we find an object that is over the file size
limit.

# How does it work?

The basic flow of this hook is as follows:

1. Read stdin line by line for push information (`old-ref new-ref refname`)
1. For every line you receive get a list of files that have changed.
1. Inspect each file's object and get its size in bytes.
1. Compare that size to a limit that you have set.
1. Provide useful feedback to the user about any exceptions that are rasied.
1. Exit accordingly

We also need to be aware of a couple of exceptional cases: branch creation and
branch deletion. On a branch creation the `old-ref` will be a null SHA1
(`0000000000000000000000000000000000000000`). Similarly on a branch deletion
the `new-ref` will be a null SHA1. For the purposes of this hook, there should
be nothing happening in a branch deletion that is of any importance to us, so we
will want to skip looking at those types of operations.

For this example I will use a bash script which calls out to the git utilities.
I am choosing to do it this way so that we can discuss how each step works, and
what is happening. Similar hooks could be written using modules designed to
interact with git, such as [GitPython][gitpython] or [Grit][grit].

# What does the script look like?

{% gist 5827183 %}

# Breakdown
Let's explore the script a little bit, and go through the operations happening
to make all this work. The first git operation, which is at line 21 is
performing a [`git-diff`][gitdiff] with several options.

* `--stat` is making diff display a diffstat
* `--name-only` makes the output only contain the name of the files changed
* `--diff-filter=ACMRT` limits the output of the diff to only contain additive
  operations.
* `${oldref}..${newref}` generates the diff based on changes to be applied to
  oldref by newref.

The next git operation is on line 23, where [`git cat-file`][gitcatfile] is used
to obtain the object size of a file in a commit. When we encounter a file that
is above our defined MAXSIZE (line 27), the script outputs some actionable
information to the user. When the filesize condition is met, the EXIT variable
is set to a non-zero value, and we exit with that at the end of execution.

# Conclusion
As you can see, it's trivial to put together a simple bash script that can
enforce file size restrictions on pushes. Alternatively, it would be a
relatively simple modification to allow this script to operate on total commit
size. Knowing the core concepts required to perform a task like this makes
it easier to port it to using a git module in a different scripting language.

[githooks]: https://www.kernel.org/pub/software/scm/git/docs/githooks.html
[gitdiff]: https://www.kernel.org/pub/software/scm/git/docs/git-diff.html
[gitcatfile]: https://www.kernel.org/pub/software/scm/git/docs/git-cat-file.html
[gitpython]: http://gitorious.org/git-python
[grit]: https://github.com/mojombo/grit
[llo]: {% post_url 2013-06-13-locating-large-objects %}
