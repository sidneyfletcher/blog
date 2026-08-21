+++
title = "Understanding LLMs From Scratch: Tokenizers"
date = 2026-08-20
description = "A first-principles tour of tokenizers, subwords, BPE, pretokenization, and the performance work behind Gigatoken."
+++

You wake up, Rip Van Winkle-like, from a deep sleep. It's 2026 and LLM's are taking over the world. Time to learn something about how these work, you think (and write up your notes).

Best to start at the very beginning of the LLM processing pipeline - what is a tokenizer and how does it work?

## What are Tokenizers?

At its most basic, a tokenizer is a way to take a string of text and turn it into a list of numbers that a computer can understand.

As an English speaker, the first thing that comes to mind is to break up the text into words and just assign every word an ID (e.g. a->1, the->2 etc.). There's a couple problems with this approach:

1. There's far too many words for a model to handle - how do you cut the vocabulary down to a reasonable size for a model?
2. What do you do about rare words or new words? Angiogenesis? Skibidi Toilet? What is your model to do, is it giving UNK?
3. How do you encode the relationship between words? How do you deal with "encode" vs "encodes" (i.e. inflection)? How do you deal with capitalization? Contractions? Hyphens?

Historically search engines used two techniques: stemming and lemmatization. (The way I remember the difference is that stemming will turn both "stemming" and "lemming" into "stemm" and "lemm" but lemmatization should leave "lemming" alone).

Another approach people tried was character-level tokenization (inspired by the pixel-by-pixel image approaches du jour). The problem with these approaches is that they required models to reconstruct the structure of words (i.e. their morphology) which is sort of annoying.

## The Subword Compromise

The approach that modern LLM"s landed on was a middle ground between words and characters: they tokenize subwords.

Consider the phrase "subword morphology". We can break this down into its various affixes: "sub-" meaning under, "word", "morph-" meaning shape, "-ology" meaning study of. Because these subwords carry meaning, these could potentially make good tokens!

So here's the basic recipe for modern LLM tokenization:

1. Preprocessing (split out reserved tokens, NFC normalization)
2. Pretokenization (splitting the text into fragments via regex)
3. Tokenization (byte pair encoding with a learned vocabulary)

The byte-pair encoding works roughly as follows: if you start with a word like "encoding", consider all single characters as tokens, then look at each pair of tokens (e.g. e+n, n+c etc) and merge the pair in the vocab that has the merge order. Repeat this until you run out of tokens to merge.

Eventually what you're left with is a list of IDs ([1, 56, 1083]) that you can turn into embeddings and feed to the model.

This works pretty well for English and English-like languages to capture some amount of subword morphology using empirical stats. (as a side note, I am curious why teams working on Chinese use BPE/UTF-8 bytes? I'm not a Mandarin speaker at all but I thought Chinese character glyphs have the equivalent of subwords - is it possible to capture that? Not frequent enough? Han unification weirdness?)

In answer to our 3 questions above, this does pretty well:

1. it reduces the vocabulary to a fixed size based on empirical data
2. it tokenizes unknown words successfully by breaking them down into hopefully semantic subwords
3. it sometimes captures recurring morphological pieces (although interestingly English contractions are mostly handled inside pretokenization)

All in all clever, no?

## Understanding the performance of tokenization

Tokenization is generally not the slow part of the LLM pipeline. But as someone who suffers from a mild case of C++ brain, I couldn't resist looking at the performance when I saw a cool project optimizing it: https://github.com/marcelroed/gigatoken/.

Here's what the project documents: the pretokenization step (which generally uses a regex to split the text) is commonly the slowest step in tokenization. The project uses SIMD & branchless programming techniques to handwrite the pretokenizer (as well as some other cool techniques).

This is impressive work, but the README didn't really explain the performance things that I wanted to know. My brief notes from poking a bit at the code:

1. It's true that pretokenization is the bottleneck but only in a proximate sense. The BPE algorithm itself is slow and non-local and annoying to optimize (e.g. it's not the intuitive algorithm where you greedily take the longest matching prefix). The hard boundaries that come from pretokenization are net positive to the performance of the program.
2. The pretokenization step includes a series of English contractions (e.g. 's) - kind of unclear why these are inside pretokenization instead of being reserved words?
3. Other than that, the pretokenization is essentially SIMD code splitting up runs of Unicode letters, numbers, whitespace etc followed by a couple fixup rules related to whitespace. The nice thing about this formulation is that it offers a clear way to handle incremental tokenization, look at the previous character class run and apply the fixup! This is basically a more complicated version of our original whitespace splitting.

This is my basic understanding of tokenizers? Lemme know if I've gotten something wrong!

Next time, I'll try to understand positional embeddings.
