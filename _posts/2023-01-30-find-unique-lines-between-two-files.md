---
layout: post
title: Find Unique Lines Between Two Files
categories: tips
tags: shell
---

Today I needed to find the lines that were present in one file but not in another. The files weren't sorted.

The context was that I had noticed I had a new subscriber to my mailing list but couldn't figure out who it was. So I exported the list prior to the sign-up, and the list after the sign-up, and wanted to see which line in the second file was new.

I found an easy solution that I wanted to log here for at least my own future reference. It came in the form of the `comm` command, and it was as simple as this:

```bash
comm -13 <(sort file1) <(sort file2)
```

In brief, the `comm` command can be used to compare two sorted files line by line and display the lines that are unique to each file. It also displays lines if they exist in both files.

The output is organized into three columns: (1) lines unique to the first file, (2) lines unique to the second file, and (3) lines that exist in both files.

The `-13` passed to `comm` suppresses columns 1 and 3, so that only column 2, containing the lines unique to the second file, is displayed.

The `<` reads the files as standard input, which allows us to pass it the output of the `sort` command. This makes sure that `comm` only gets sorted input, which it needs in order to be able to compare the files line by line.

It's a nifty little shortcut. Hopefully you find this as useful as I did. There are lots of potential uses for `comm` so it's good to be familiar with how it works.
