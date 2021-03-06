SPDX-FileCopyrightText: 2016-2020 Graham R. Cobb
SPDX-License-Identifier: GPL-3.0-or-later

# extents-lists

Programs to manipulate lists of file extents.

The main motivation for these scripts is to understand more about the usage of shared extents,
and hence disk usage, on my btrfs disks.
However, they are technically valid on any filesystem which supports the FIEMAP ioctl.

The scripts manipulate lists of occupied filesystem extents, treated as just sets of blocks.
The scripts do not track which files the extents relate to, just which extents are in use.

These scripts allow answering questions like "how much space am I wasting by keeping historical snapshots",
"how much data is being shared between two subvolumes", "how much of the data in my latest snapshot is
unique to that snapshot" and "how much space would I actually free up if I removed (just) these particular 
directories".

## Program summary

The following commands are available:

```
extents-list [-s] <find-options> >output-list

extents-merge [-s] <input-list >output-list

extents-size [<blocksize>] <input-list

extents-union [-s] <filename>... >output-list

extents-intersection [-s] <filename>... >output-list

extents-difference [-s] <filename>... >output-list

extents-expr [-s] <directory>|@<file>... [<operator> <directory>|@<file>...]... >output-list

extents-to-remove [-s] <disk> <directory>... >output-list
```

## Extent lists

The basic datastructure is an extent list.  This is a text file with one line for each in-use
range of blocks (identified by the *physical_offset* displayed using `filefrag -v`).
The line has three fields (in decimal, space separated): starting offset, ending offset and length (in blocks).
The scripts all write that format, but they ignore the length field and anything else on the line when
reading the files. The file **MUST** be in numerical order by starting offset and the extents **MUST
NOT** overlap.  Adjacent extents **SHOULD** be combined. Note that `extents-merge` will fix files that
do not follow those rules.

## extents-list

```
extents-list [-s] <find-options>...
```

*find-options* are options which will be passed to the `find`
command to generate a list of files.

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

Examples:

* extents-list some-directory/some-file
* extents-list some-directory
* extents-list subvolume-directory -mount

## extents-merge

Unsorted list of extent start and finish pairs on stdin.
Output list with overlapping and adjacent extents merged.

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## extents-size

```
extents-size [<blocksize>]
```

Calculate space occupied by the extent list provided on stdin.
Blocksize defaults to 4096.

## extents-union

```
extents-union [-s] <filename>...
```

Output the extents contained in **any** of the named extents lists.

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## extents-intersection

```
extents-intersection [-s] <filename>...
```

Output the extents contained in **all** of the named extents lists.

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## extents-difference

```
extents-difference [-s] <filename>...
```

Output the extents contained in the first named file which are **not** present in any of the other files.

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## extents-expr
```
extents-expr [-s] <directory>|@<file>... [<operator> <directory>|@<file>...]...
```

Very simple expression evaluator for extents.
Extents lists are read from each existing *file* (if the @<file> form is used)
or generated automatically from each *directory* and the operators are applied.
The default operator used between adjacent terms is "union".

Operators are:
* (plus sign) + union
* (caret) ^ intersection
* (minus sign) - difference

There is no operator precedence, no parentheses and all evaluation is strictly left-to-right.
Note: **THIS MIGHT CHANGE** at some time in the future.
This means that using wildcards in the righthand-side
of an operator will almost certainly not achieve the desired effect (see example below).

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## extents-to-remove
```
extents-to-remove [-s] <disk> <directory>...
```

Generates the extent list of extents which will be removed from the disk if the named
directories are removed.
Directory names must be **relative** to the named disk.

Note: this script may take a very long time as the extents list for the whole disk must be generated!

If the -s option is specified then the resulting list is automatically sent to `extents-size`
instead of being sent to *stdout*.

## Examples

* To find out how much space is being wasted by keeping historical snapshots
of a subvolume called *cobb* you could use:
```
 	extents-expr -s cobb.snapshot.* - cobb
```

* To find out how much data is shared between two specific snapshots use:
```
	extents-expr -s Media.snapshot.20160801T030601+0100 ^ Media.snapshot.20160815T000602+0100
```

* To determine how much of the data is not in any of the existing snapshots
is harder and and cannot be done in a single `extents-expr` command.
```
	extents-expr -s cobb - cobb.snapshot.*
```
does **not** give the desired result because of the strict left-to-right evaluation of the expression.
The actual effect is to subtract the first of the cobb.snapshot.* directories and then add in all the rest.

The way to do this is to use a temporary file.
For example:
```
	extents-list cobb.snapshot.* >/tmp/cobb.snapshots.extents
	extents-expr -s cobb - @/tmp/cobb.snapshots.extents
```

* To find out how much space particular files/directories/subvolumes/snapshots are occupying you could use:
```
	extents-list -s some/file some/directory some/other/directory
```
But this counts space which might be shared with other files,
so does not tell you how much space would be released if the files were deleted.

* To actually find out how much space would be still be used if particular files/directories/subvolumes/snapshots are removed
requires measuring the space occupied by the files which would **remain**.
The key is to work out the `find` options which return those remaining files,
remembering that extents-list always adds `-type f -print0`
to the end (which means that a `-o` may be needed at the end of the options).
In the example below, the disk is mounted as */mnt/data* and the two directories to be removed are
*some/directory* and *some/other/directory*:
```
	extents-list -s /mnt/data -path /mnt/data/some/directory -prune -o -path /mnt/data/some/other/directory -prune -o
```

* To create an extent list referring to the extents which would actually be removed in the example above
requires generating the extent list for the files/directories being removed and
subtracting the extents for the remaining files (generated above) from it:
```
	# Generate list of extents which would remain
	extents-list /mnt/data -path /mnt/data/some/directory -prune -o -path /mnt/data/some/other/directory -prune -o >/tmp/remaining.extents
	# Calculate extents to actually be removed
	extents-expr /mnt/data/some/directory /mnt/data/some/other/directory - @/tmp/remaining.extents
```

* The simple case of removing whole directory trees from the disk can be automated using `extents-to-remove`.
This script requires that the files to be removed are directory trees (as in the two examples above), not a more complex
set of files (such as date-based or filename-based).
```
	extents-to-remove -s /mnt/data some/directory some/other/directory
```

Be warned: the last three examples take a very LONG TIME (and require a lot of space in $TMPDIR)
as they effectively have to get the file extents for almost every file on the disk (and sort them multiple times).
They take over 12 hours on my system!

## Notices
Copyright (c) 2016-2020 Graham R. Cobb.
This software is distributed under the GPL 3.0 (see the copyright notices and the LICENSE file).

`extents-lists` is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 3.0 of the License, or
(at your option) any later version.

`extents-list` is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
