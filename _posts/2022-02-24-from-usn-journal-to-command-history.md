---
layout: post
title: From USN Journal To Command History
tags: Forensics USN-Journal
---

In this mini blog post, we will take a look at how we can get PowerShell command history timestamps using the USN journal and ConsoleHost_history.txt file.

## USN journal? ConsoleHost_history.txt?
USN journal (Update Sequence Number Journal) or Change Journal is something like a diary for files and directories in the NTFS filesystem, it keeps writing about what happened to its files and directories, and the journey of the files (also directories) taken to become what it is now.

<div style="text-align: center">
    <img src="/assets/images/from-usn-journal-to-command-history/diary.png" width=550 >
</div>

Well, that’s USN Journal, what about ConsoleHost_history.txt file? what is it?

It is a simple text file, which each command you entered in PowerShell will append to it.
You get the last command you entered when you press the up arrow key because of this file.
And it’s located at: `%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\`

<div style="text-align: center">
    <img src="/assets/images/from-usn-journal-to-command-history/example.png" width=750 >
</div>

## Let’s mix them
I put together a simple diagram for the overall process.

<div style="text-align: center">
    <img src="/assets/images/from-usn-journal-to-command-history/diagram.png" width=750 >
</div>

Let me explain it.

When you enter a command in PowerShell and hit the enter, PowerShell will append entered command to the ConsoleHost_history.txt file.

Obviously, this append causes a change in the file. when we say change USN journal comes into play and stores the change as a new record.

And each record contains FileName, USN, TimeStamp, Reason, …

<div style="text-align: center">
    <img src="/assets/images/from-usn-journal-to-command-history/example-2.png" width=750>
    <figcaption>Example of a USN journal record</figcaption>
</div>

Every command we enter takes the same manner.
As a result, we will have a list of records containing timestamps and a list of commands, which we can correlate.

Using power forensics command I created a small script that helps you to parse and extract the commands and their timestamps.

You can get it from this this [gist](https://gist.github.com/Cyber-Aku/bc060ed41b1c5db512f3048ad2960e1d).

<div style="text-align: center">
    <img src="/assets/images/from-usn-journal-to-command-history/example-1.png" width=750 >
</div>
