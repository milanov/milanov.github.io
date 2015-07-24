---
layout: post
title:  "Statistics Over Git Repositories"
date:   2015-07-24 14:30:00
tags: version-control git statistics
image: /assets/article_images/2015-07-24-statistics-over-git-repositories/git-log.png
---

Inspired by the first ever Destroy All Software [screencast](https://www.destroyallsoftware.com/screencasts/catalog/statistics-over-git-repositories){:rel='nofollow' target='_blank'} (and shamelessly copying its title) I've also decided to use the shell and the git command line tools to iterate over revisions, computing a statistic for each revision. In particular, the task I have in mind is to rank commits based on their **message length** to **lines changed** ratio. This will prove to be an interesting ranking, showing on the one spectrum small changes that have needed large explanations and on the other short messages that explain massive edits.

First we'll start by using [git rev-list](http://git-scm.com/docs/git-rev-list){:rel='nofollow' target='_blank'} in a bash for-loop to list all the commits that are reachable by following a given commit or commit range (assuming we're in a git project directory).
{% highlight bash %}
for commit in $(git rev-list $1); do
	#use $commit here
done
{% endhighlight %}

Next we need to find the commit message length and number of altered lines for each commit. The former can be done using [git log](http://git-scm.com/docs/git-log){:rel='nofollow' target='_blank'} in combination with the `--format=%B` option which specifies what we want only the whole commit message. The latter can be achieved using the same command, but with an empty format argument `--format=` to skip everything from the output and the `--shortstat` flag to display statistics about changed lines. The `wc` and `awk` commands are used in addition to the git-ones to get the commit message length and sum the added/deleted lines respectively.
{% highlight bash %}
git log -n 1 --format=%B $commit | wc -m
git log -n 1 --format= --shortstat $commit | awk '{print $4 + $6}'
{% endhighlight %}

What remains is to only divide these two metrics for each commit and sort by the obtained ratio. Here's how the whole bash script looks:
{% highlight bash %}
#!/bin/bash

set -e

for commit in $(git rev-list --no-merges $1); do
	length=$(git log -n 1 --format=%B $commit | wc -m)
	changes=$(git log -n 1 --format= --shortstat $commit |\
		awk '{print $4 + $6}')

	ratio=$(echo "$length/$changes" | bc -l)
	echo "$ratio $commit"
done | sort -n
{% endhighlight %}

Note that names are shortened and a line is split so that the whole codeblock could fit better on the page. The only unexplained commands and options are the `--no-merges` flag for `git rev-list`, which is quite self-explainatory, and the ratio calcucation using `bc -l` instead of a regular division. This is done because bash does not support floating point numbers and the easiest way in my opinion is to use `bc` with the `-l  --mathlib      use the predefined math routines` option.

To put this script in use, I ran it on one of my university projects, namely a [Language Workbench Challenge](http://www.languageworkbenches.net/past-editions/){:rel='nofollow' target='_blank'}, available on [Github](https://github.com/milanov/QL){:target='_blank'}. The project is relatively small, but was done in a team and has a decent number of commits. The commit that comes on top of the chart is [this one](https://github.com/milanov/QL/commit/f82dcc87876fb1a6016d71a0613874d4c3a58ded){:target='_blank'}, in which I've changed a total of **6 lines** (one of which is an addition of an empty line) and have explained it in a total of **251 characters**. On the other end we have [a commit](https://github.com/milanov/QL/commit/86f195b8e98263033165ed7dd2c280af9c543ee9){:target='_blank'} with **514 additions** and **509 deletions** clarified in **49 characters**. 

Running it on a more real-world project like the [express framework](https://github.com/strongloop/express/){:rel='nofollow' target='_blank'} leads to [this commit](https://github.com/strongloop/express/commit/0fdceb3de379923ea8087b1f1cc9e6382e6ef41d){:rel='nofollow' target='_blank'} (albeit producing a few `Runtime error (func=(main), adr=6): Divide by zero` exceptions in the meantime). In it the author basically changes a **single line** while explaining it in **437 characters**.

While the results of the script may not be exactly useful, it is still a good and fun exercise in **bash scripting** and **git command line** tooling.