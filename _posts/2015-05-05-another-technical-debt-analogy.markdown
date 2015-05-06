---
layout: post
title:  "Another Technical Debt Analogy"
date:   2015-05-05 14:30:00
tags: programming technical-debt metaphor
image: /assets/article_images/2015-05-05-another-technical-debt-analogy/files-and-folders.jpg
---

The Technical Debt metaphor (or the [unhedged call option](http://www.higherorderlogic.com/2010/07/bad-code-isnt-technical-debt-its-an-unhedged-call-option/){:rel='nofollow' target='_blank'} one, if you prefer it) is used to describe what can happen when some part of the software work is postponed in order to deliver other parts faster.

>In this metaphor, doing things the quick and dirty way sets us up with a technical debt, which is similar to a financial debt. Like a financial debt, the technical debt incurs interest payments, which come in the form of the extra effort that we have to do in future development because of the quick and dirty design choice. We can choose to continue paying the interest, or we can pay down the principal by refactoring the quick and dirty design into the better design. Although it costs to pay down the principal, we gain by reduced interest payments in the future.
>
> &mdash; <cite>[Martin Fowler](http://martinfowler.com/bliki/TechnicalDebt.html){:rel='nofollow' target='_blank'}</cite>

Every time I've had to do a clean operating system install and restore data from backup I get reminded of how **poorly structured** I've made it. The same observation holds for bookmarks, email filter folders, etc. Most people, including me, start arranging and saving files in nicely named folders like *Documents*, *Downloads*, *Pictures*, *Projects*. As time goes by, however, projects unintentionally end up checked out in *Downloads* or *Home*, photos end up in a subfolder of the *Desktop* folder, books and papers in a sub-sub-folder of *Projects* and so on. This is how huge unstructured folders named *Desktop 1*, *Desktop 2*, etc get created. These small but **continuous misplacements** of files and folders resembles in some detail the process of accumulating technical debt.

{:.center}
![Slightly misplaced bookmarks](/assets/article_images/2015-05-05-another-technical-debt-analogy/misplaced-bookmarks.png "Slightly misplaced bookmarks")

In software development, this debt is "paid" every time a developer sits down to write code and is **annoyingly slowed down**. In the files and folders metaphor, it is paid when searching or traversing the filesystem. And while it may seem easy to find your own files by searching by name, for example, you may not always remeber it. The same practice of structuring code poorly and then wasting time to find what you need in a huge codebase is, however, usually rightly rejected in the software industry.

Continuing the metaphor, some people are good at getting used to their personal files/folders organization and can, although sometimes with hassles, find what they need. The same holds for some developers. Not implementing a method or class properly or misplacing it and then using it to fix a bug or implement a feature may **feel not that bad**. Such developers may be very familiar with the codebase and may be accustomed to the oddities they've introduced. Getting new developers exposed to these oddities would be similar to asking a random person to find something in your own personal file/folder structure.

This analogy is probably not perfect. Some people even [question](http://blogs.perl.org/users/ovid/2011/11/technical-debt-when-metaphors-go-wrong.html){:rel='nofollow' target='_blank'} the original metaphor. Restructuring a filesystem may take a couple of hours, while rewriting a project may take years. The mental effort in the two activities is far from the same. Technical debt is accumulated not only by misplacing code, but by bad design, time-pressure, lack of knowledge and etc. However, the analogy could serve as a good practical example for developers who haven't been exposed to **the debt** or still don't grasp its concept.