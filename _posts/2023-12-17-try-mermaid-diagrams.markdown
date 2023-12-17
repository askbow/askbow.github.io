---
layout: post
title:  "Moving site to Jekyll, trying Mermaid diagrams, Mathjax LaTex"
date:   2023-12-17 09:30:00 +0100
categories: sandbox
mermaid: true
math: true
---
# 🐱‍👤🐱‍👤🐱‍👤

## MD?

I hope to later find a good MD editor for Jekyl. Something sensible.

> Note: I collapsed the older MD post into this one; no need for more than one sandbox

## 🧜‍♀️ Mermaid?

Mermaid is a javascript library to draw diagrams directly in Markdown

https://mermaid.live/

### A test diagram

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```


## Mathjax? LaTeX?

Mathjax is a framework that implements LaTeX rendering for the web. I use it to display nice formulas directly from Markdown.

https://jbergknoff.github.io/mathjax-sandbox/

### A simple formula

A formula can be rendered inline: $y = a\times x^2 + b\times x + c$

Or as a block:

$$L = \frac{\pi^2\times R}{2} $$