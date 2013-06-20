---
layout: post
title:  "Blocking large file commits in git"
date:   2013-06-18 09:51:24 -0700 
categories: git
---

In my previous blog post on [Locating large objects in a git
repository][llogr], I covered some of the git utilities that can be used to
identify large files in your repository. In this post I'd like to go over how
to create a server-side hook that will block large files before they get into
your repo. Many of the same utilities from the post referenced above will be
used in creating this hook.

# What are server-side hooks?
[Git hooks][githooks] allow you to perform actions at different points in the git workflow. Server-side hooks can be executed at certain specific phases of the push process:

# Which hook type should I use?

# What utilities should I use?

# How does it work?

# Conclusion

[githooks]: https://www.kernel.org/pub/software/scm/git/docs/githooks.html
[llogr]: {% post_url 2013-06-13-locating-large-objects %}
