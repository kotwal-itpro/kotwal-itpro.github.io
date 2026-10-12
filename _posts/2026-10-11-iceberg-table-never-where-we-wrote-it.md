---
layout: post
date: 2026-10-11
title: "The Iceberg Table That Was Never Where We Wrote It"
description: "What I found while adding Apache Iceberg support to Apache Hop: data saved in the temp folder, bugs that only showed up in a real install, and what the maintainers caught in review."
---

For the past week I've been working on [Apache Hop](https://hop.apache.org/), an open-source tool for building data pipelines. My goal was to let Hop read and write [Apache Iceberg](https://iceberg.apache.org/) tables on its own, without needing a Spark cluster.

That work is now merged into Hop's main branch and will be in the next release. Along the way I ran into several bugs. Most of them had one thing in common: every test passed while the bug was there. This post is about those bugs, how I found them, and what I'd check for next time.

## Some background

If you haven't used Iceberg: it's a way of storing a table as a set of ordinary files, usually Parquet files, in a folder or a cloud bucket. Next to the data files, Iceberg keeps metadata files that say which data files belong to the table right now. When you add rows, Iceberg writes new data files and then a new metadata file. That last step is called a *commit*. Until the commit happens, nobody sees the new rows.

Hop could already work with Iceberg, but only when a pipeline runs on Spark. Many Hop users run pipelines on Hop's own engine instead, on a laptop or a single server. So I proposed adding Iceberg support there too. The maintainers agreed on one rule, which I liked: there should be one "Lake table input" and one "Lake table output" that work the same way on both engines, so a table written by one can be read by the other. One of them, Hans Van Akelyen, put it as "design once, run everywhere".

I split the work into four pull requests: moving the existing code into a shared plugin, reading, writing, and docs.

## Bug 1: The table was in the temp folder

If both engines are supposed to work on the same tables, I wanted a test that proves it. So I wrote one: write a table with the Spark engine, then read it with Hop's own engine.

It failed right away. Hop's engine said there was no table at that path.

That was odd, because Spark's own tests, which write a table and then read it back, all passed. So I went looking for the files. The table did exist. It just wasn't where we wrote it. A pipeline told to write to

```
/tmp/hop-it-iceberg/0005-table
```

had actually written to

```
$TMPDIR/hop-iceberg-path-catalog-warehouse/file:/tmp/hop-it-iceberg/0005-table/
```

That's a folder inside the system's temp directory, with a subfolder literally named `file:`.

**Why it happened.** Iceberg tables usually live in a *catalog*, a service that maps table names to folders. Iceberg can also work with a plain folder path. But in Spark, it only treats your text as a path if you read the table in one particular way. Hop was passing the path in a way that Spark treated as a table *name* instead. So the catalog did what catalogs do with a name: it created the table in its own storage folder, which happened to be in the temp directory.

Reading used the same name, so it looked in the same wrong place and found the table. That's why every write-then-read test passed. The data was fine until the operating system cleaned out its temp folder.

**The fix.** I reported this as [issue #8794](https://github.com/apache/hop/issues/8794). The fix, [#8795](https://github.com/apache/hop/pull/8795), gives each folder its own small catalog that stores tables in that folder, so the table now lands exactly at the path you asked for.

Matt Casters, who started Hop, reviewed the fix twice before merging it. Both times he found cases I had missed:

- a table at the top of a Windows drive, like `C:\orders`, was rejected;
- two ways of writing the same folder, like `file:/x` and `file:///x`, or `C:` and `c:`, were treated as two different places;
- a path with a space in it ended up in a folder called `my%20lake` instead of `my lake`.

Each of those has a test now.

**What I learned.** A test that writes data and reads it back only proves the system can find its own data again. It doesn't prove the data is where you think it is. The test that would have caught this is tiny: after writing, check that the table's files exist at the path you gave.

## Bug 2: Every test passed, but every real write failed

My new code had plenty of unit tests, including pipelines that write with several copies in parallel and pipelines that fail on purpose to check that nothing gets committed. They all passed.

Then I built a full Hop install and ran the same pipeline the way a user would. Every write failed with:

```
Format model is not registered for format PARQUET and type interface org.apache.iceberg.data.Record
```

**Why it happened.** When Iceberg starts up, it looks for the code that knows how to write Parquet files and registers it. It does this once. To find that code, it asks the current thread which set of classes it can see.

In a unit test, everything is in one place, so Iceberg finds what it needs. In a real Hop install, each plugin has its own separate set of classes, and the threads that run a pipeline don't see inside the plugins. So Iceberg looked, found nothing, and from then on believed there was no Parquet writer at all.

**The fix.** A small helper that points the thread at the plugin's classes whenever Hop calls into Iceberg.

**What I learned.** If your code runs inside a plugin system, some of your tests have to run inside that plugin system too. Unit tests alone can't show this kind of problem.

## Bug 3: Some rows twice, some not at all

Hop can run a step as several copies in parallel to go faster. For reading, that means splitting the table's files between the copies, so each piece is read by exactly one copy.

My first version split the work by position in a list: copy 1 takes pieces 1, 3, 5 and so on, copy 2 takes 2, 4, 6. My test expected 1,200 rows and got 1,202. Some rows came through twice and some were missing.

**Why it happened.** Each copy asks Iceberg for the list of pieces on its own. Iceberg builds that list by reading several files at the same time, so the order can be different every time you ask. Copy 1 and copy 2 got the list in different orders, so "every other piece" meant different pieces for each of them.

**The fix.** Instead of using the position in the list, each piece is assigned to a copy based on its file name and where it starts in the file. Every copy works out the same answer no matter what order the list comes in.

Bart Maertens from the Hop project committee found the next problem in the same area while reviewing this. Each copy was also choosing on its own which version of the table to read. If someone wrote to the table in between, two copies could read two different versions. Now all copies agree on one version first. Before merging, Bart also tested the whole thing himself against a table in S3-style cloud storage.

**What I learned.** Don't depend on an order that nobody promised you, especially when several processes need to agree with each other.

## Getting the commit right

This part starts with a design decision, and then a few bugs in the details.

**The decision.** When a pipeline writes to a table, when should the commit happen? If each parallel copy committed on its own, someone reading the table could see half of a run. If the commit happened as soon as the write step was done, a later step in the same pipeline could still fail, and the table would already contain data from a failed run.

So Hop commits only once, after the whole pipeline has finished without errors. All the files written in that run become one new version of the table. If anything fails, the files are deleted and the table stays as it was. If the table didn't exist yet, it's created in that same commit, so a failed run doesn't even leave an empty table behind.

**The details.** Matt reviewed the write pull request, [#8819](https://github.com/apache/hop/pull/8819), three times in one day. Each round found something real.

*Two writers, one lost.* For tables stored as a plain folder, a commit means writing the next numbered metadata file, like `v5.metadata.json`. If two pipelines write at the same moment, only one of them may get number 5. The other has to notice and try again. I used a file "move" for this, but as Matt pointed out, that move quietly replaces the file if it already exists. Both pipelines would think they had succeeded, and one of their changes would be gone. I wrote a test with several writers at once, and on the old code, changes really did go missing. Now, on a local disk, the commit uses a file operation that fails if the file is already there. On cloud storage like S3 there's no such operation, so the docs say to keep to one writer at a time there, or to use a catalog.

*"Finished" too early.* Hop marks a pipeline as finished before it gets to my commit code. A scheduler watching for "finished" could start the next job before the new data was actually in the table. Now the commit runs first. The test for this watches the pipeline's status from a separate thread while three copies write 3,000 rows. On the old code, it saw "finished" before the data was there on every single run.

*Overwriting an empty table.* When a pipeline overwrites a table, it first checks that nobody else changed the table since the run started. But that check was skipped when the table was still empty. So if someone added the first rows during an overwrite, those rows would be wiped out. Now the check always runs.

None of these show up in a test that runs one pipeline at a time. All of them could lose someone's data.

## What I'd check next time

- **Check where the data actually is**, not only that you can read it back.
- **Test in the real setup**: a full install with its plugins, not only unit tests.
- **Don't trust an order** that the API doesn't promise.
- **Commit when the whole job is done**, and report "done" only after the commit.
- **Test with several writers at once** if your code will ever have several writers.

And get a careful reviewer if you can. Matt Casters and Bart Maertens found problems I would not have found on my own. Thanks to both of them.

The code is in Apache Hop's main branch: reading in [#8802](https://github.com/apache/hop/pull/8802), writing in [#8819](https://github.com/apache/hop/pull/8819), and the docs in [#8859](https://github.com/apache/hop/pull/8859), which is still being reviewed.
