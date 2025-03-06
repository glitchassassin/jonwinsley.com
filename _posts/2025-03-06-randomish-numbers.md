---
layout: post
title: "Generating Randomish Sequential Numbers"
date: 2025-03-06 13:50:00
author: Jon Winsley
comments: true
summary: I needed to generate visually distinct IDs from sequential identifiers. It turns out that a Hull-Dobell linear congruential generator is far less scary than it sounds.
---

I'm working on a project for a client where a fleet of mobile devices (which may be offline) print out paper tickets, each of which has to have a unique identifier. These ticket numbers need to be human-friendly in case something goes wrong and the ticket needs to be entered manually, read off over the phone, etc.

There are immediately a number of challenges:

1. We need to guarantee uniqueness without a central authority (the mobile devices might have to generate a ticket ID without a connection to the server)
2. We need to make the ticket numbers human-friendly (short, characters are not easily confused, easy to read over the phone or punch into a tablet)
3. We need to make the ticket numbers visually distinct (so they don't *look* sequential, which might cause confusion around adjacent tickets)

## Guaranteeing Uniqueness

This is, fortunately, fairly easy: we can combine a sequential device ID (issued by the central server when the device is linked) and a sequential ticket ID (managed locally per device).

Now, device `001` can create tickets `0001`, `0002`, etc. and the ticket `0010001` will never conflict with device `002`'s ticket `0020001`.

But if we use numbers, we'll have to make IDs longer. We could easily have more than 1,000 device registrations (though 10,000 is unlikely). And it's unlikely, but possible, that we could have more than 10,000 tickets created on a device. So the ticket number would actually look like `0001000001`.

That's getting pretty long. Human working memory does okay with around 7 items, so this is pushing it.

## Human-Friendly Ticket Numbers

Fortunately, we have options. Zooko O'Whielacronx published the [human-oriented base-32 encoding](https://philzimmermann.com/docs/human-oriented-base-32-encoding.txt) which fits our purposes well: more characters than just digits, but only lowercase to make it easier to read over the phone, and some confusing characters (`0`, `l`, `v`, and `2`) have been eliminated.

This means we can encode our device counter with three characters `yyy` and our ticket counter with four characters `yyyy`, fitting nicely in that 7-item target.

This is getting closer, but the sequential repetition isn't ideal:

```
yybyyyy
yybyyyb
yybyyyn
...
```

## Random-ish Sequences

What we really want is a way to map a sequence of numbers (1, 2, 3...) to an apparently random sequence (57, 1095, 2) while guaranteeing the outputs will never collide (so 10 and 27 can't both map to 998, for example).

After a bit of digging (and consulting with Claude, which gave me some helpful keywords to search for but failed at implementing a solution), I came across the [Hull-Dobell variant of the linear congruential generator algorithm](https://en.wikipedia.org/wiki/Linear_congruential_generator#m_a_power_of_2,_c_%E2%89%A0_0).

This is an intimidating name for a *really* simple piece of code. It's a kind of pseudo-random generator: you start it with a seed number, and then use the result as the seed for the next iteration. It is guaranteed not to repeat for `m` iterations.

```ts
/**
 * Randomizes a number in the space of 32^<power> (where <power> is 1 or greater)
 */
export function lcgRandom(seed: number, power: number): number {
  const m = Math.pow(32, power);
  // a - 1 is divisible by all prime factors of `m` (which will always be 2)
  // a - 1 is divisible by 4 if `m` is divisible by 4 (which will always be true)
  // `5` always meets these criteria
  const a = 5;
  // c is coprime to m (only common factor is 1), so 1 always meets these criteria
  const c = 1;

  // Ensure seed is within valid range
  seed = (seed + m) % m;

  // Apply LCG formula: X_{n+1} = (a * X_n + c) % m
  return (a * seed + c) % m;
}
```

Instead of tracking a ticket counter value locally, we'll just track the last value, and use it as the seed for the next value when we need a new ticket:

```ts
export async function getNextTicketCounter() {
  const seed = await database.localStorage.get<number>(TICKET_RANDOM_SEED);
  const counter = lcgRandom(seed, 4);
  await database.localStorage.set<number>(TICKET_RANDOM_SEED, counter);
  return counter;
}
```

## Mission Accomplished

Now our tickets are unique, easy to communicate "out-of-band," and easy to distinguish. Perfect:

```
coindx1
coiktp5
coiwzfe
...
```