---
date: '2025-09-12T20:14:49+02:00'
title: 'The resemblence of complex large scale systems and biology'
type: 'posts'
draft: true
---

I recently watched Hank Green’s vlog [**“The Hardest Problem Evolution Ever Solved”**](https://www.youtube.com/watch?v=On2V_L9jwS4) about the evolution of land animals, more specifically land vertabrates. Hank was really enthusiastic about how neofunctionalism works – the mechanism in which existing functionality is adapted to create new functions. Distinct examples are gas bladders turning into lungs, stress coping proteins that we now call crystallin and are used in our eye lenses or carotene doubling as waterproof skin so our cells can keep their inner *"seawater"* environment. All examples of neofunctionalism. Nature uses whatever is lying around, adapts it, and if it kinda works, keeps iterating.

That video stuck with me because it *felt* familiar.

### Software systems are complex systems that evolve over time

All the microservice codebases I've seen that felt well made followed some unified approach, one set of pattern, but teams adapted those to solve their specific problems.

But every now and then there is a challenge that is special, or different. Searching your product catalog or authenticating a user are quite specific and often use specific technologies like a search engine or a JWT to fulfill their function in an efficient way. Those require new ideas, but build on the same building blocks. Still use microservices, gRPC endpoints. Maybe there is an external system like Elasticsearch that we use. But we also have the common boring things like adding a new field to all user profiles might technically be quite similar to adding a field to all content pages.

### The architecting nightmare

Biology never produced a creature by whiteboarding the perfect organ first and yet is able to create astonishingly reliable "systems". When working with other clients I often see a need to create very specific idea of how every detail has to look like. Often this comes paired with an urge for "beauty", before even having anything that works. Way too often I then see that this puts a burden on the engineers or that problems are solved in really complicated ways making it hard to implement, maintain or even understand. If you pair this with client organizations you can get quite wild solutions that lose touch with the actual problem they want to solve

There is a spectrum when it comes to what work software architects do. I once worked in a project where they defined general guidelines like

 * inter-domain communication is done via Kafka
 * we use gRPC inside the cluster, REST to the outside
 * no caching

which every team could challenge when they had good reasons. In some cases it simply made sense to introduce caching or to call an endpoint from a different domain. But those cases should be brought to our "guild" to discuss whether the ideas behind the guidelines where maybe misunderstood or whether it was a valid approach in this particular example or maybe more generally. This felt organic and served more to foster a more uniform approach then to dictate solutions. We had our building blocks and used them to build everything we need, adapting it in the few places that needed it, comparable to neofunctionalism, don't stray to far from what is known.

On the other side of the spectrum are the "here is your system diagram" approaches which always feel like keeping control. This to me is the worst kind of approach to organize a project. If the engineering teams do not make and own their decisions this will result in frankenstein systems driven by design and not by need. On top of that it simply builds the wrong engineering mindset in the teams.

### Three guardrails I keep repeating to myself

1. **Start end to end, not perfect**: Make sure to verify in the real world, deploy to prod as often as possible that beats a gorgeous diagram every time.
2. **Let pain drive refactors**: Align your optimizations along the main pain points. Imaginary issues often simply waste time.
3. **Small & familiar → cheap to swap**: Approach your goals step by step. The simpler each piece and every step, the cheaper the changes will be and there will be changes.

### Ship the slice, watch it evolve

The only way I know to keep complexity from eating us alive is to release an end‑to‑end slice early, observe how it doesn't match all the ugly unforeseen edge cases, and slice again. Force yourself to release something, nature also doesn't build something new hidden from the world only to release it when it is perfect.

Use the boring cloud primitives, lean on libraries that already exist – preferably standard libraries, and reserve novelty for the genuine gaps. Complexity will still emerge. Software is weird and requirements mutate, but if each little service stays simple, you can adapt.
