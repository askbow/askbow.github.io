---
layout: post
title:  "Explain Like I'm 5: ChatGPT"
date:   2025-05-02 09:30:00 +0100
categories: musings
mermaid: false
math: false
---
# Explain Like I'm 5: ChatGPT

Scrolling morning Reddit, I stumbled upon a great question in the ELI5 sub.
It went something like this:
>Why doesn't ChatGPT admit it doesn't know stuff?

Most top answers there were correct, but alas more like high-school or higher level.

It got me thinking. Can I explain it to a literal five year old?
Now, five-year-olds in the year 2025 are probably way smarter than I was.
Here's an explanation that would have worked for my own self, as far as I was aware of things at that age.

>What follows is a really basic analogy. As any model, an analogy has its limits.

# ELI5: How do they "train" a "model"?
Imagine, a rabbit hopping on a field 🐇 ⛳⛳⛳
The field is uneven: little hills🏞, little valleys🌄⛺

We plant on the field various grasses 🌾🌺🌻🌼🌷 the rabbit might like to chew or hide in.
We also place little rocks to create paths, place food 🥕🌽 and water 🌊 so that the rabbit could go to these things.
All-in-all, we made a really huge garden for our rabbit to roam.

We first guide the rabbit through the garden by some path. We show it some food and some water. Then, we let it roam free.

It starts hopping around. Depending on how playful it is, it might even jump over our little stone fences to other areas! 
It goes on some journey around our garden. We record its path on our map🗺.


Maybe the path we have drawn on the map is not what we'd like to see. We want the rabbit's journey to make a pretty picture on our map.

We carefully move the things around in the garden. We hope to guide the rabbit's journey more to our liking.

We repeat the experiment many times, a million millions times. Until the rabbit's journey on the map looks pretty.

The rabbit itself doesn't see our map. It just choses where to hop next. And it can only hop from the point where it is to some other nearby point that it sees and want to get to.

## How does this translate?

- The rabbit hopping around carelessly is the LLM's algorithm.
We can have slightly different algorithms by picking other animals!
-  The placement of the things in the garden are called the "model weights". The weights that made a rabbit's path pretty may not work for a horse🐎! A horse and a rabbit share some fundamentals. They are both mammals, have four legs, eat grass, etc. But they differ in development and movement ability. The LLM algorithms share the common fundamentals of the neural nets, generators, and the attention feedback loop.
- The initial path by which we guide the rabbit before setting it free is called the "prompt".
- The training makes the path look "pretty", just like the model training made the produced text look like it was written by a person.

# ELI5: How do people interact with a "model"?

We make the whole thing into a farm!

We let other people visit our farm and play with the animals on their respective fields.

For example, our visitors can take turns introducing a rabbit to the field, to guide it in, and then let it roam free. Then it hops around a bit.
Then they guide it some more, until the rabbit's journey drawn on a map looks pretty to them.


But it turns out, the initial guiding path is important. So we take over guiding the rabbit the first few steps into the garden, before letting our visitors take it further.

## How does this translate?

- people can choose different underlying models, like they can choose which animal to play with
- people write their question to the model, the same way we let them guide the rabbit into the field
- after letting the rabbit roam free for some time, people can take guide it a little more, adding to their interaction -- but the whole rabbit's journey is recorded; the same way you respond to ChatGPT's initial reply if it wasn't exactly what you wanted
- when we take over the initial few steps, we create what is called the "system prompt"

# ELI5: Why doesn't it stop and say it doesn't "know"?

The rabbit is careless and doesn't know anything. The beauty is in the eye of the beholder. The figures we draw on our map as we trace the rabbit's path look nice to us, but the rabbit doesn't really see the map. It just chooses where to hop next.
And it always does hop somewhere in the garden, attracted by the things it sees that we placed there.

## How does this translate?

- an LLM operates on symbols connected by a graph, and all it does is finds some pseudo-random path on that graph.
- I like an explanation I've read from Kent Back recently, for coding LLMs. There is a set of all possible programs. The LLM doesn't know which ones are actually correct. In a more general way, an LLM finds a nice-looking path in a field of all possible vectors of tokens. It always goes along **some** path.


# ELI5: What do the LLM researchers and developers do?

It's a lot of work.

Some people are herding and nurturing the rabbits, the horses, and the cats. Somebody has to breed those special fluffy chickens too! Some people try to make it wore with little mice and hampsters.

Other people are doing some landscape design so that the animals roam the gardens in specific ways. There are other people who make the shovels and excavator machines for the gardeners.

Yet other people, create the initial paths that are later useful for the guests.
Then some other people build the farm and post the ads.

## How does this translate?

- some people design and optimize the algorithms to a variety of performance requirements
- some people run the training to produce the model weights for various needs
- some people make the hardware that runs the computation
- some people develop the prompts, set up the infrastructure (backend, frontent), etc