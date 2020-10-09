---
title: Eth vs Fabric - Deep Dive Comparison - Part 1
date: 2020-10-01 00:00:00 +05:30
tags: [about]
description: An attempt to deconstruct the similarities and dissimilarities between Hyperledger Fabric Architecture and Ethereum Architecture
---

# Eth vs Fab - A Deep dive Comparison - Part 1

> **Caveat Emptor** - If you are looking for a non-technical comparison, this article is not for you. But if you are a ninja ⚔️ developer or a fledgeling 🐦 one, this is the right place to understand the internals of the platforms side-by-side.  

In this article, we deconstruct some of the similarities and differences between two of the most popular Blockchain Platforms - Hyperledger Fabric v2 and Ethereum v1 (as of 10th Oct 2020). The motive is simple. We have much material explaining the workings of each of these ledger platforms. However, there still does not exist a deep-dive comparison of these ledger platforms. 🧐

Most comparisons that I have stumbled is either a tabular one discussing the core features of the platforms like Consensus, Permissioning, smart contract language, and the likes. Personally, as a technology geek and a person who likes to dive deep and ask a lot of questions, these articles do not quench my thirst. 😅🍺

So, as an Architect, I decided to make an attempt to understand these platforms deep and explores the fundamentals that make these Platforms the most acclaimed ones. 

This will be a six-part series of articles where I will attempt to take each of the fundamental concepts of a Blockchain Platform and attempt to expound them as it is architected in each of the platforms with the rationale behind it. I will also try to provide an essence of the key architectural considerations made while designing the platforms. 

I will try to explore the following areas, and in each, I try to cover three fundamental aspects. First is the need, second is key architectural considerations and finally the comparison. My goal is to be objective and non-partisan. For this, I will share the references to the sources from where I have obtained and compiled the information. 

## Comparison Areas

| Area | Covered In |
|------|------------|
|Overall Architecture|Part 1|
|Block Architecture|Part 2|
|Ledger & Storage|Part 2|
|Identity Management|Part 3|
|Transaction Lifecycle|Part 4|
|Smart Contracts & Ledger State|Part 4|
|Consensus Process|Part 5|
|Security|Part 6|
|Confidentiality|Part 6|


<hr/>

# Overall Architecture of Hyperledger Fabric vs Ethereum

In this section I attempt to break down the fundamental cogs that drive the Fabric Blockchain Operating system from the Ethereum Blockchain Operating system. Although mouthful it may sound, both of these platforms are irrefutable distributed in nature.