![](assets/images/groxio-logo@0.5x.png)
# Technical Glossary
### Version: Very, very draft

contents:

* Design Principles
* Things That Compose


# Design Principles

## Principle: Single Clause Description

The basic idea is to design each “thing” to have a single
responsibility, where “thing” can be a function, a module, a component,
an assembly, and so on.

To help folks understand this, I suggest using the _single clause_ rule:

> Whenever thinking about writing a “thing,” first describe it in a
> single (human language) sentence. Be as complete and accurate as
> possible.
>
> Then split that sentence into its clauses (a good approach is to split
> it at commas and the word "and").
>
> Each clause now describes an independent "thing."

### SCD---example 1

> Calculate the total of the order by looking at each item, multiplying
> the quantity by the price, then adding the item totals, and the adding
> in sales tax on top of that total.

This may seem contrived, but I've seen this function implemented.

The clauses here are (in a more logical order)

* given an item containing a quantity and a price, calculate its total
  price
* given a list of items, sum their net total price
* given a net total, calculate its sales tax
* sum the net total and sales tax to get a final price

The benefits of this split are clear: as the world changes, and we get
special requirements (such as quantity discounts, perhaps), we now have
clear places to make these changes. We have also moved into a more
_function passing style_ (see below).

### SCD---example 2

> Add a new user to the system and if they are a manager set up
> departmental access rights.

Here we have two clauses:

* add a new user
* if they are a manager set up access rights

although the second clause probably breaks down into two more:

* determine if a user is a manager
* set up departmental access rights for a user

Implementation of this depends to some extent on factors not covered by
the description. In particular, we don't know if the creation of the
user and the setting of access rights must happen atomically.

If not, we could write our three functions as:

~~~ elixir
user_details
|> create_user()
|> maybe_add_departmental_access(&user_is_manager?/1)
~~~

If it was transactional, then that adds a new clause the the
description, and our implementation could be:

~~~ elixir
user_details
|> build_changeset()
|> maybe_add_departmental_access(&user_is_manager?/1)
|> commit()
~~~

### SCD---example 3

> Players access the noughts and crosses site and are paired up to play
> a game.

"Ah," we say. "A Phoenix app."

"But, no!" responds SCD. "It's two Phoenix apps"

* pair two people who want to play a game
* play a game given two players

One application is the front door to the game. It has a "find someone to
play with" button. When clicked, it either adds this browser to a
waiting room or, if there's already another browser in there it sends
both to the second application.

The second application receives a request (via redirect) from each
browser. The request contains a token which is the same for each,
allowing a game to be formed.

Each Phoenix app is then decomposed into components recursively.

One benefit of this approach is that we now have a rendevous app for _any_
n-player web game.

It's also a model for implementing things such as SSO.


### SCD---example 4

> Accept a four digit guess for the current mastermind game and return
> the resulting score”

This is an interesting example, because this approach would seem to be
totally natural.

~~~ elixir
guess |> apply_to(game) |> score
~~~

But the reality is that game is likely to have internal state, which
means that the actual API would be more like

~~~ elixir
{ score, new_game } = apply_to(guess, game)
~~~

So it’s clear that the “and” in the description is mirrored by in the
implementation: the function both updates the game state and returns a
score.

Split it:

> Update a game by accepting a four digit guess

~~~ elixir
game = game |> update_with(guess)
~~~

> Return the current score from a game

~~~ elixir
score = score_of_last_move(game)
~~~

This design is an example of the _Query XOR Update_ principle, described
next.

## Principle: Query XOR Update

## Principle: Function Passing Style

# Things That Compose
