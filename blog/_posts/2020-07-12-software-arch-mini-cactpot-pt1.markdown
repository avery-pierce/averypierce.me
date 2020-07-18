---
layout: post
title:  "Software architecture study: mini cactpot pt I"
date:   2020-07-12 10:36:00 -0500
categories: software-architecture swift
---

[Mini cactpot](https://na.finalfantasyxiv.com/lodestone/playguide/contentsguide/goldsaucer/cactpot/) is a chance-based game that resembles a scratch-off ticket. The game consists of a 3x3 grid containing the digits 1 through 9, and each digit is used exactly once. The player chooses a horizontal, vertical, or diagonal row of 3 squares. The sum of these squares is used to determine the player's payout, according to a pay table which is shown to the player. However, at the start of the game only 1 random square is revealed to the player. The player may then choose 3 more squares to reveal before picking a row.

It is possible to compute the probability of winning a certain amount from a certain row, given a partially-revealed card. Armed with this information, a savvy player can always choose the row that maximizes their chances of winning the jackpot. In this study, I will create a software module written in Swift that can compute the average payout for each row in a partially-completed mini cactpot ticket. 

<div class="note">
<p>This is not a programming tutorial. The goal of this post is not to teach someone to write code, it's an exercise for experienced programmers to sharpen their skills by examining trade-offs in a program's software architecture.</p>
</div>

## Ticket

The most obvious component of the system is the cactpot ticket itself. The programmer should be able to create a mini cactpot ticket, and set the value of any cell in an expressive way. In addition, the programmer will want to query for all the cells in a given horizontal, vertical, or diagonal row without any fuss. But the ticket itself is not responsible for computing probabilities. It's only responsible for getting and setting state.

Let's examine how we could model this in Swift. There are 9 cells. Each cell is either empty, or contains a digit 1-9. My instinct is to model this as the type `[Int?]`. This model isn't a perfect representation of our data – the array will permit more than 9 entries, and it will permit entries outisde the range of valid inputs. Therefore, it may be a good idea to make this internal state private to the ticket type.

Swift has two options for complex data types that support `private` internal variables: `struct` or `class`. My hueristic for choosing between these two is this: If I change one of the properties of this type, would I consider it to be fundamentally distinct from the original? If so, it's a `struct`. If not, it's `class`. For example, consider a type `Color`, with `r`, `g`, and `b` properties. If I change one of those properties significantly most people would say that I've created an entirely distinct `Color`. Therefore, `Color` is a `struct`. Now consider a type `PaintBucket`, which has a `color` and a `paintRemaining`. If I `addDye()` to the bucket to change the `color`, most people would say the the bucket itself remains unchanged. Even if I `pourPaint()` to lower the `paintRemaining`, the `PaintBucket` is not distinct. Therefore `PaintBucket` is a `class`.



### Workspace

We could constrain the range of inputs by using a custom enum to define each valid digit, but since we still need to find the sum of digits, each case of our enum would have to map to an `Int` value anyway. As for the array length, we could throw an error if the programmer attempts to 



I think the natural way to model this is to create an array of 9 `Int?`. Swift doesn't support fixed-width arrays, and it doesn't have a way to declare that only the integers 1 through 9 are permitted in this type, so `Int?` is the least complex option.

