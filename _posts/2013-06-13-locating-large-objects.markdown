---
layout: post
title:  "Locating large objects in a git repository"
date:   2013-06-13 14:11:20
categories: git
---

If you manage [`git`][git] repositories it isn't uncommon to run into a problem
where someone, maybe even yourself, commits a large file into a repository.
Especially in repositories that have a large number of committers, you may not
have visibility into all branches or changes that are being committed.  There
are several commands that are part of `git` that will allow you to locate these
large files/commits. 

NOTE: All instructions below will assume that you are working in a bare repo,
not a cloned copy. If you don't have access to the repository on the server,
run your clone with `--mirror` to get a local copy of the bare repo.

# How do I get a list of commits for a repo?
If you're reasonably certain that there was a recent commit that added this new
large file, a quick way to identify it is using [`git rev-list`][rev-list]. From
within the bare repo you would run something like the following:

{% highlight bash %}
$ git rev-list --all --pretty=oneline --since={1.month.ago}
bb440b353b637a4fbab5e1c482982268abc71cf4 file23
637223b0c27a0653ec13e51a685e5e0f02a86cda removing a file
c709809f6f82bcc523f076ad994eaeea7a39b5ee testfile4
0d69f430382f3836381815cc6e2765a94650f39a testfile3
beca3d64743b097a01877b7d37035e0be98faaea lfdkjskldfjs
537cab9218f543a86af893c6780fbe11a7a22cee sdfkjsdkfs
24f59c5d5da941f4008b5637db5737d9cb4a85da asdfkjsdkf
54edada19006eaab6592dc93a8280ba092b4802c Branched deep file
f8bd89de7f5333dbd11246751fb051215c4dfc7a Another big file!
a3fc750255d97f6f35db4b3bfa14081c038e3304 v
{% endhighlight %}

I'll breakdown the command above for anyone not familiar with using `git rev-list`:

- `--all` tells git to operate on all refs in the `refs/` directory.
- `--pretty=oneline` condenses the output to one line to make it easier to
  digest.
- `--since={1.month.ago}` tells git to limit the output to commits that are
  less than 1 month old.

The two column output above consists of the SHA1 and subject of all matching
commits. The bit we're especially interested in is the SHA1 for the commit, the
subject may or may not have any relevant information to our search.

# How do I get blobs from a commit?
Now that we have a list of commits, with their SHA1's, we can use another git utility,
[`git diff-tree`][diff-tree]. This will compare the content of blobs between
trees. Based on the output above if I ran:

{% highlight bash %}
$ git diff-tree -M -C -r -c --no-commit-id 54edada19006eaab6592dc93a8280ba092b4802c
54edada19006eaab6592dc93a8280ba092b4802c
:000000 100644 0000000000000000000000000000000000000000 e6556c4a65229fbad1b07ce36f5eeac42c069a4f A  1/2/3/4/5/testfile2
{% endhighlight %}

With the options to `git diff-tree` above, we're asking it to track
copies/renames, display all changes from the commit's parents, traverse into
subtrees, and suppress the commit id. I won't go in to what all the output here
means, but mainly we're interested in columns 4-6. These represent SHA1 of a
blob object, the operation type, and the path to the file.

# How do I get the sizes of blobs?
Now that we have the SHA1 of a blob we can get its size using [`git
cat-file`][cat-file].

{% highlight bash %}
$ git cat-file -s e6556c4a65229fbad1b07ce36f5eeac42c069a4f
10240000
{% endhighlight %}

We've passed the `-s` option to `git cat-file` to have it output the size of
the object in bytes. The object in question is 10240000 bytes, or ~10MB. This
may or may not be the file we're interested in, but we did manage to get the
file size of a blob from a particular commit.

# How do I find which branches contain the offending commit?
We can also use the SHA1 of the commit from earler to find out what branch or
branches this blob is referenced in. For that we use [`git branch`][branch] as
follows:

{% highlight bash %}
$ git branch --contains 54edada19006eaab6592dc93a8280ba092b4802c
    gmac_test
{% endhighlight %}

Knowing which branches are effected can be important if you decide that you need
to purge the repo of this large file (something I'll cover in a later blog
post).

# Putting it all together
Now that we know how to pull the relevant information out of git, a simple
script can be written to walk all the commits, inspect their blobs, and display
files that meet or exceed a certain size.

{% highlight bash %}
#!/bin/bash

GIT_CMD=/usr/local/bin/git
RANGE=$1
MAX_SIZE=$2

# Iterate over a list of commits
for commit in $(${GIT_CMD} rev-list --all --since={${RANGE}} --pretty=oneline | cut -d' ' -f1); do
  # Iterate over that commit's blobs
  for diffout in $(${GIT_CMD} diff-tree -r -c -M -C --no-commit-id ${commit} | awk '{print $4":"$6}'); do
    blob=$(echo ${diffout} | cut -d':' -f1)
    filename=$(echo ${diffout} | cut -d':' -f2)
    # Skip if this is a file deletion
    if [ "$blob" = "0000000000000000000000000000000000000000" ]; then
       continue
    fi
    # Get the blob size
    blob_size=$(${GIT_CMD} cat-file -s ${blob})
    # Compare it to MAX_SIZE
    if [ "${blob_size}" -gt "${MAX_SIZE}" ]; then
      echo "${commit} ${filename} ${blob_size}"
    fi
  done
done
{% endhighlight %}

It certainly isn't the prettiest scipt, but in a pinch it would yield a good
amount of data to continue digging with. When run against my test repository
this is the output I received:

    $ bash ../find_large_files.sh "1.month.ago" 90000
    bb440b353b637a4fbab5e1c482982268abc71cf4 1/2/3/4/file23 10240000
    c709809f6f82bcc523f076ad994eaeea7a39b5ee testfile4 10240000
    0d69f430382f3836381815cc6e2765a94650f39a testfile3 10240000
    537cab9218f543a86af893c6780fbe11a7a22cee testfile2 10240000
    24f59c5d5da941f4008b5637db5737d9cb4a85da testfile3 10240000
    54edada19006eaab6592dc93a8280ba092b4802c 1/2/3/4/5/testfile2 10240000
    f8bd89de7f5333dbd11246751fb051215c4dfc7a testfile2 10240000

The output fields are: `<commit SHA1> <file path> <size in bytes>`.
Additionally, the output could be piped to `sort -nk3` to  sort on filesize.
Alternatively, you could sum up the blobs for each individual commit, in case
the issue isn't one large file, but several smaller ones.

As mentioned before, you can take the commit SHA1 and pass it to `git branch
--contains <SHA1>` to see which branches this large file may have been merged
into.

# Conclusion
There are many utilities that ship with `git` that allow you to see certain
pieces of information about your repository's object store. Using these tools
in concert can yield data that would otherwise be challenging to obtain.
There will be a followup blog post to this one about writing a server side git
hook to prevent these large files from ever being committed to your repository.
That hook will make use of some of the same utilities that were covered in this post.

I hope this has been helpful to anyone who has found themselves trying to
troubleshoot unwarranted repository growth. Feedback is certainly welcome, if
there's an easier way to do what I've described here I'd be excited to hear
about it.

[git]: https://github.com/git/git
[rev-list]: https://www.kernel.org/pub/software/scm/git/docs/git-rev-list.html
[diff-tree]: https://www.kernel.org/pub/software/scm/git/docs/git-diff-tree.html
[cat-file]: https://www.kernel.org/pub/software/scm/git/docs/git-cat-file.html
[branch]: https://www.kernel.org/pub/software/scm/git/docs/git-branch.html
