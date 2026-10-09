---
layout: post
date: 2026-10-09
title: "Five Ways to Run the Same AI Job, Three Kinds of Mistakes, and What Broke at 100,000 Records"
description: "Round two of an open test of batch AI jobs: a self-hosted model, Claude, 100,000 records, and a quota error that wasn't about quota."
---


Two days ago I wrote about [giving two AI models the same instructions](/2026/10/07/same-instructions-opposite-mistakes/) and watching them get things wrong in opposite ways. Since then I've added two more ways of running the job, made it 100 times bigger, and run it again on different days. Here's what changed, and what didn't.

## The setup, quickly

Same job as before. I have fake customer support chats, and for each one the AI has to say which product it's about, what kind of problem it is, how the customer feels, and whether they asked to be contacted again. I wrote the chats, so I know every right answer.

This time I ran it five ways:

- **OpenAI** (gpt-4.1-mini)
- **Google Gemini** (gemini-3.5-flash-lite)
- **Anthropic Claude** (Haiku 5.5), with its normal settings
- **Claude again**, with its "thinking" turned off
- **A model I ran myself**: Qwen2.5-7B, an open model, on a GPU I rented for a few minutes

The first four are "batch" services. You send everything at once and collect the answers later, for about half the usual price. The last one has no queue at all. It's just a computer doing the work.

## Three kinds of mistakes

All five got the product and the problem type right every time. Nearly all of them got the follow-up question right too, once I'd explained it properly (that was the big lesson last time).

The "how does the customer feel" question is where they split, into three groups:

**Too negative.** OpenAI (93% right) and my self-hosted model (85% right). A customer writes "My laptop stopped working after three days. Let me know what you need from me." That's a calm, helpful message about a problem. Both called the customer unhappy. My own model did this twice as often, and sometimes even turned happy customers into unhappy ones.

**Too forgiving.** Gemini (82% right). "This is really frustrating" came back as neutral.

**Depends on one setting.** Claude scored about the same both ways, 89% and 91%. With thinking on, its mistakes were split about evenly between too harsh and too forgiving. With thinking off, almost all of them were too harsh, like OpenAI's. Same model, same instructions, one switch, and the balance of its mistakes changed. The two settings gave different answers on about 1 chat in 9.

If you take one thing from this post: **a score doesn't tell you which way a model is wrong.** If something downstream counts unhappy customers, two models with similar scores can give you very different counts.

## Running it myself: fast and cheap, with a catch

The self-hosted model surprised me. Once it was set up, it did all 1,000 chats in **29 seconds**. The hosted services took 3 to 10 minutes, because you wait in their queue. It also cost about **$0.004 per 1,000 chats** in GPU time, about a tenth of the cheapest hosted option.

The catch is the setup. Installing the software, downloading the model and getting it ready took about 8 minutes on the rented GPU, which cost more than the actual work. And it was the least accurate of the five on feelings. It's also an older, smaller model than the hosted ones, so that isn't the last word on self-hosting.

## What happened at 100,000 records

Then I made the job 100 times bigger: 100,000 chats, on each of the three batch services.

**Nothing went missing.** All three sent back all 100,000 answers. None needed a retry, and the scores matched the smaller runs almost exactly. I'd half expected the trouble to start here. It didn't.

**The waiting changed a lot.** Gemini did it in 10 pieces of 10,000, about 4 minutes each, so 45 minutes in total. Claude took all 100,000 in one go and needed 2 hours and 44 minutes. OpenAI was the odd one. Its pieces took 14 minutes early in the afternoon, then slowed to almost an hour and a half by the evening. Same job, same size, six times slower depending on when I sent it.

The price per 1,000 chats didn't change with size: about 9 cents on OpenAI and Gemini, 5 cents on Claude.

**The only real failure was on my side.** Eight hours into the OpenAI run, my internet connection dropped while uploading the ninth piece, and the job stopped. This is exactly what I built the tool for. It keeps a record of what's done, so when I ran the same command again it picked up the last 20,000 chats and didn't resend, or pay for, the 80,000 that were already finished. It now also waits and retries when the connection drops, instead of giving up.

One thing went wrong before any work started. I sent Gemini all 100,000 chats as one job. It took five minutes to upload, then Gemini said:

> You exceeded your current quota, please check your plan and billing details.

That sounds like a billing problem. It wasn't. When I sent the same work in pieces of 10,000, one after another, every piece went straight through. The real problem was that one job was too big for how much work my account can have waiting. The message just didn't say so.

That one changed my tool. It now recognizes a "queue full" answer, waits or sends smaller pieces, and doesn't count it as a failed attempt for the records involved.

## Does it give the same answer twice?

This was the question I most wanted answered. I sent the exact same 1,000 chats, with the same instructions, to each service on three days in a row: Wednesday, Thursday and Friday.

- **OpenAI** gave the same answer all three days on **99.3%** of chats. Only 7 changed.
- **Claude** did it on **94.4%**. 56 changed.
- **Gemini** did it on **87.5%**. 125 changed, about 1 in 8, even with its randomness setting turned all the way down.

Here's the strange part. The overall scores barely moved from day to day: Gemini's "how does the customer feel" score was 82%, 81% and 81%. But underneath, a different set of chats was right each day. Every single chat that changed was answered correctly on one day and wrongly on another. The services wobble on the borderline cases, and which side a case lands on changes from day to day.

If you compare this week's numbers with last week's, some of the change is the model, not your customers. Know how big that wobble is before you read anything into a small shift.

## What I'd tell someone starting out

1. **Explain every field in one plain sentence.** It took one question from 31% right to nearly 100%.
2. **Check every record off.** Don't trust a job that says "done". Count what came back.
3. **Find out if your model "thinks".** Thinking uses up the same room as the answer. Claude ran out of room on 1% of chats until I turned it off.
4. **Send big jobs in pieces, and keep a record of what's done.** Don't take "quota exceeded" at face value, and expect the network to drop at least once in a long job.
5. **Don't expect the same answer twice.** Even with randomness turned down, some answers change between runs.
6. **Before switching models, compare the mistakes, not the scores.**

I've written these up properly in a [guide](https://github.com/kotwal-itpro/steadybatch/blob/main/docs/GUIDE.md). The tool is now on PyPI (`pip install steadybatch`), and the code, the fake chats and every result are on GitHub at [kotwal-itpro/steadybatch](https://github.com/kotwal-itpro/steadybatch). If you want to cite it, it now has a DOI: [10.5281/zenodo.23221956](https://doi.org/10.5281/zenodo.23221956).
