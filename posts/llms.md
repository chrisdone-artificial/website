---
date: 2025-04-18
title: LLMs
tags: ai
---

# 2025-06-04

Small update: I'd presently describe my outlook as sceptical. I go in
cycles, of long troughs of scepticism, with very, very, brief
(one-day), widely spread out, spikes of FOMO and belief that AI is an
existential situation for software developers (see previous 2025-04-18
thought dump). But those are becoming more infrequent. I think I see
which way the trend for me is going.

Having cut out the final vestige of social media (RSS feeds; BazQux)
from sheer disinterest, and digital devices in the evening entirely, I
am also serene and a little bored by the whole thing. But it still
does play on the minds of some friends.

I might change my mind on the next update, we'll see.

# 2025-04-18

I've been collecting thoughts on LLMs in a peacemeal way. I add to this document from time to time. It's not an article as such.

* Hype: On forums like HN people have long, familiar, arguments about whether AI is hyped or not.
  This is consistent with any other hyped thing, like crypto or micro services or whatever.
* Objective value: Tools have gotten very good at generating code, which means that some people's work,
  especially those which involve generating throwaway things which are iterated on quickly, has been accelerated,
  and some tasks which were previously inaccessible to people who didn’t have the patience to wait through poorly
  documented APIs is now accessible.
* Not learning: By looking at any topic which is not sufficiently exposed in the training data, one can see that, for example,
  an individual human can learn a given topic and it doesn’t take millions of examples to learn a thing,
  because we can use reasoning --symbolic reasoning/manipulation-- to learn a program language which has just one book, for example.
  Whereas the LLM doesn’t actually learn the programing language, but essentially is learning how to parrot it as if it’s a human language.
  This is why all the popular models are kind of crap at Haskell and cannot speak Lojban at all,
  despite there being a complete and comprehensive book describing Lojban's grammar, semantics, dictionaries and everything like that.
* If it’s not written, it doesn’t exist: Essentially, if someone hasn’t written about it in detail, and posted it online or in a book,
  then it’s basically a piece of human insight which is not accessible to an LLM.
  For example, if nobody ever talked about print debugging online or in books,
  and then you asked an LLM about different ways to debug a program, I'm quite confident the answer would
  not mention any concept of printf debugging because that is a completely human experience which is interactive and comes from experience.
* Objective benefit: LLMs essentially turn human language into code which is digestible, analysable, translatable, summarizeable and generable.
  That means we can apply all of our regular engineering tools like compression and linting and test suites and DSLs. Modulo nondeterminism.
* Objective limitation: As implemented,
  we know that LLMs are capable within their probabilistic transformers to estimate answers to questions, but,
  for example, cannot count the individual letters in a word or do basic arithmetic.
  (At the time of writing; these leaks will be plugged eventually by postprocessing and tools, I'm sure.)
* Speculation on anthropomorphism: I think the reason that LLMs appeared to reason is that the base model,
 which is essentially a snapshot of the Internet, is then of course trained on millions of conversations in
  which a question is posed and an answer is generated,
  which can give the appearance and impression of comprehension and an internal mental model. Clever hans.
* Relating to language theory: I’m reminded of a chapter in Stephen Pinker‘s book The Blank Slate, in which he discusses
  the case of a young woman who has a developmental issue such that she is not able to reason or function independently,
  but is able to speak and tell long, detailed stories, which are completely fabricated,
  but which have an internally consistent grammar and a consistent storyline.
  In this chapter Pinker is pointing out that you can have a grasp of language, but no grasp of logic reasoning et cetera.
* Relation to a consistent pattern of abstraction due to layers of complexity:
 the MIT book Structure and Interpretation of Computer Programs was switched from a from-the-ground-up, principles first,
  implementation of various topics, to essentially a goodie bag of scientific ways of using vast APIs and Python.
  And this is completely not a criticism of Abelson or Sussman: in fact, this reflects a deep insight;
  that they had the foresight and integrity to accurately reflect in the education.
  Hardware was too complex; now it’s been abstracted away. The operating system is a
  layer of abstraction, and the programming language is a layer of abstraction.
  And now there is a language which is not only a black box, but it’s also probabilistic... so it’s yet another layer of unpredictability.
* Dijkstra on anthropomorphism:
    * People talking about “asking it..” and “it thinks…” etc. makes me feel icky.
    * Quote from EWD854: I think anthropomorphism is worst of all. I have now seen programs "trying to do things", "wanting to do things", "believing things to be true", "knowing things" etc. Don't be so naive as to believe that this use of language is harmless. It invites the programmer to identify himself with the execution of the program and almost forces upon him the use of operational semantics.
* The Lisp Curse: see [my other article](https://chrisdone.com/posts/the-llm-curse)
* A prior colleague remarked, "Category theory is white magic, AI is black magic." It stuck with me as it succinctly summarises much of the above.
* Humans as machines: Dijkstra and Frank Herbert have this in common: Dijkstra railed against anthropomorphising machines, but he also
  argued against treating humans in mechnistic terms, i.e. [treating them like machines.](https://chrisdone.com/posts/ai-machine-thinking) It's a funny way to flip things around.
