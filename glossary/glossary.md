![](../assets/images/groxio-logo@0.5x.png)
# Technical Glossary
### Version: Very, very draft

contents:

* Things That Compose
* Design Principles


# Things That Compose

### transformation

Take some data and map it to different data. May be implemented by a
function, a map, or some external authority.

### Function

Code that performs a transformation. The SCD principle (later) tells s
that a function shold be highly focussed on a single task.

### Modules

A coherent collection of functions. Given that functions are
transformations, this implies that the functions in a typical module
will be working on related data.

### Components

A free-standing collection of one or more modules. Again, a component will
be focussed on one task.

In Elixir terms, a component is what you generate with `mix new`

### Assembly

An assembly is a collection of components that together deliver end-user
value.

Typically this is the unit of deployment.

## Stateful and Stateless Modules and Components

State is a property of a module or component (MoC). If an MoC has state,
then the results of calling an API function may change depending on previous
invocations of some API functions.

### Stateful

A stateful component _always_ holds that state in one or more separate
processes. This may be in the form of an external agent-like process, or
it may be implemented locally in a traditional process recursive loop.

A component that works on state, but that passes that state out to its
clients, which then pass it back on subsequent calls, is _not_ stateful.
Instead, it is forcing its clients to be.

### Stateless

A stateless MoC will not create processes in order to
hold state, and all its code will run within the process(es) of its caller.

Such MoCs are often called _libraries_.

### What Constitutes State

We have to be careful when talking about state, because the boundary
between stateless and stateful can be blurry.

For example, we may have a pool of workers that do database transactions
for us. Each worker is created with a database connection, which it
retains for its lifetime. In that sense, the worker has state.

However, a client of the worker sees a stateless component, in that
there's no memory between one invocation and the next---a client API
call does not visibly affect any subsequent client API call. I'd
therefore categorize such workers as stateless.


# Design Principles

The fundamental idea is to create designs that favor composition, both
of data and functions. The key to this is keeping things focussed and small.

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
> the quantity by the price, then adding the item totals, and adding
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

(This applies to functions in stateful MoCs only)

A call to any API should either

* cause an update to the state of the MoC, or
* return a value based on that state

It should never do both.

(This could also be called "Tell Or Ask" :)

## Principle: Function Passing Style

When the behavior of a function changes depending on some flag-like
value passed to it, then that function is doing two things. Split the
variable behavior out into two or more separate functions, and then pass
those functions instead of a flag.

So, rather than

~~~ elixir
def write_formatted_message(msg, to_console?) do
  msg = format(msg)
  if to_console? do
    IO.puts(msg)
  else
    Logger.info(msg)
  end
end
~~~

so

~~~ elixir
def format_and_write(msg, writer) do
  msg
  |> format()
  |> writer.()
end

def writer_console(msg), do: IO.puts(msg)
def writer_logger(msg),  do: Logger.info(msg)
~~~
