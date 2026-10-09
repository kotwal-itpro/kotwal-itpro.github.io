---
layout: post
title: "I Gave Two AI Models the Same Instructions. They Got It Wrong in Opposite Ways."
date: 2026-10-07
description: "First results from an open benchmark of batch LLM APIs: nothing broke, my first test was flawed, and OpenAI and Gemini made opposite mistakes."
---

A lot of teams now use AI models to read large piles of text: support tickets, reviews, emails. The cheapest way to do that is a "batch" job. You put all your requests in one file, send it off, and pick up the answers a few hours later. You pay about half the normal price.

I wanted to know how reliable these batch jobs really are, so I started a small open project to test them. I began with two providers, OpenAI and Google's Gemini. Here's what happened in the first round.

## What I tested

I made 1,000 fake customer support chats. Each one is a short conversation like this:

> Customer: I was charged twice for my router. Thanks.<br>
> Agent: Sorry about that. Could you confirm your order number?<br>
> Customer: Order number is A10012.

For every chat, the model had to answer four questions:

1. Which product is it about?
2. What kind of problem is it? (billing, shipping, broken product, or a "how do I" question)
3. How does the customer feel?
4. Did the customer ask to be contacted again?

Because I wrote the chats myself, I know the right answer for every one. So I can check not only whether the model replied, but whether it was right.

I sent the same 1,000 chats to OpenAI (gpt-4.1-mini) and to Gemini (gemini-3.5-flash-lite).

## The good news: nothing broke

Both services sent back all 1,000 answers. Nothing went missing, nothing came back garbled, and I didn't have to retry anything. Each job took three or four minutes. It cost about 8 or 9 cents for the whole batch on either one.

I expected at least a few problems, so this surprised me a little. My guess is that the trouble starts with much bigger jobs. That's what I'll test next.

## My first try made both models look bad

In my very first small test, I only gave the models the names of the four things I wanted, nothing more.

They nailed the first two: product and problem type were 100% right. But the follow-up question came back right only 31% of the time, on both models.

When I looked at the answers, the models weren't really wrong. They had answered a different question. I meant "did the customer ask us to get back to them?" They read it as "does this ticket need more work?" And of course every open problem needs more work, so they said yes to almost everything.

So I added one plain sentence explaining each question. That's all. The follow-up answers went from 31% right to almost 100% right, on both models.

That was the biggest lesson of the whole test. When a model seems to get something wrong, first check whether you actually told it what you meant.

## Then they made opposite mistakes

Even with the explanations, the "how does the customer feel" question was still tricky. My instructions said: judge how the customer feels about the product, and don't count the problem itself as a bad feeling.

OpenAI got this right 93% of the time, and Gemini 82%. But the scores aren't the interesting part. The interesting part is how each one got it wrong.

**OpenAI was too negative.** When a customer politely reported a problem, like "I was charged twice. Thanks.", it often called them unhappy. It reacted to the problem, even though I'd said not to.

**Gemini was too forgiving.** When a customer said "This is really frustrating", it often called them neutral. It followed my "don't count the problem" rule so strictly that it ignored real frustration too.

Same instructions, opposite mistakes.

Why does this matter? Say you count unhappy customers every week. With one model you'd count too many, and with the other you'd count too few. If you switch models, your numbers move, and it isn't because your customers changed. So when you compare models, don't just compare the scores. Look at which answers each one gets wrong.

## One real glitch

The only actual failure happened before any work started. Gemini's list of models said one older model was available. When I sent my batch job to it, Gemini refused, saying that model was "no longer available to new users". So the list and the batch service didn't agree.

That failed attempt also left my uploaded file sitting on Google's servers. I've changed my tool so it cleans up after itself when that happens.

## What's next

Over the next couple of days I'll run the exact same job again, to see whether the models give the same answers each time. After that I'll try much bigger jobs, 100,000 chats and more, and add more providers.

Everything is public: the code, the fake chats, and every result. It's all on GitHub at [kotwal-itpro/steadybatch](https://github.com/kotwal-itpro/steadybatch), including a [log of everything I found](https://github.com/kotwal-itpro/steadybatch/blob/main/docs/FINDINGS.md), my own early mistake included. If you run batch jobs like this and have seen something strange, I'd love to hear about it.

## Update, October 7: I added Claude

After writing this, I ran the same 1,000 chats through Anthropic's Claude, using its small model, Haiku 5.5. Two things stood out.

**It was the first model to lose an answer.** Out of the box, this model decides for itself when to stop and "think" before answering. Usually it doesn't. But on 10 chats it thought for so long that it ran out of room before finishing its answer. Some answers came back empty, and some were cut off halfway. My tool caught every one and asked again. Nine came back fine on the second or third try. One never did, so this run ended at 999 out of 1,000.

The room I'd given it was about eight times what a normal answer needs. It was the same limit I'd used for OpenAI and Gemini without any trouble. If you use a model like this, either give it much more room than you think it needs, or turn the thinking off.

**One setting changed its mistakes.** So I ran it again with thinking turned off. This time all 1,000 came back on the first try, in half the time: about 5 minutes instead of 10.

On "how does the customer feel", Claude scored about the same both times, 89% and 91%. That's between OpenAI and Gemini. But look at how it got things wrong:

- **With thinking on, its mistakes were split about evenly.** Sometimes too forgiving ("How do I reset my phone? This is really frustrating." came back as neutral), sometimes too harsh.
- **With thinking off, almost all of them were too harsh, like OpenAI.** "My laptop stopped working after three days. Let me know what you need from me." came back as unhappy.

Same model, same instructions, one setting changed, and the balance of its mistakes changed with it. The two runs disagreed on about 1 chat in 9.

*Correction, Oct 9: an earlier version said that with thinking on, Claude was "mostly too forgiving". Counting every mistake, they were about evenly split.*

That makes the main point of this post stronger. A score can stay the same while the answers underneath it change. If you depend on the counts, check which answers move, not just the score.

It was also the cheapest of the three: about 5 cents per 1,000 chats, against 8 to 9 cents for OpenAI and Gemini.

Next up: the same test on a model I run myself on a rented GPU, and then much bigger jobs.
