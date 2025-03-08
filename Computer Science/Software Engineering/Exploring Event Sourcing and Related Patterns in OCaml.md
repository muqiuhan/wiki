#ocaml #software-engineering #domain-modeling-driven 

> https://garba.org/posts/2016/event-sourcing/

## Intro.

The _Event Sourcing Pattern_ (Young 2010, 17; Fowler 2005a) is typically used in conjunction with the CQRS Pattern (Homer et al. 2014, 45) to form the “CQRS/ES” superpattern . Event Sourcing is about storing an application’s state as a sequence of events—that reproduce the state when replayed in the right order—as opposed to simply storing the “current state”. The reason as to why CQRS and ES go “hand in hand” is that CQRS suggests that updates should be modelled as commands, which are similar to events although they depict an intention rather than an actual outcome.

Introducing Event Sourcing creates new problems for which new solutions are required. For instance, events may be missing or wrong. In this case, the _Retroactive Event Pattern_ is proposed by Fowler (2005c). Likewise, Event Sourcing also lends itself to new interesting features such as _Parallel Models_, also proposed by Fowler (2005b).

## Business Scenario

Jane is enrolled on the Super Miles programme operated by Van Damme Airlines, headquartered in Brussels. The Super Miles programme allows passengers to upgrade to business class in exchange of 10,000 miles.

Van Damme Airlines is currently running a promotion which doubles the miles of every flight following a London (LHR) -> Brussels (BRU) connection in order to prompt passengers to take long haul flights from Brussels rather than from London.

Jane has just arrived to Bangkok (BKK) from London (LHR) via Brussels (BRU) for business—really, she is not headed to Khao San Road.

Knowing that the distance between Brussels and Bangkok is about 5,000 miles, Jane expects to have accumulated over 10,000 miles (with the Super Miles promotion) so that she can return to London in business class after so many days of ~~partying~~ business meetings.

Jane approaches the Van Damme Airline’s desk to request the upgrade to business class for her return leg back to London but she is told that she hasn’t got enough miles for that:

“Madam, your Super Miles account has only got 5730 miles.”

“That’s not possible”, Jane complains and then explains the conditions under which the Super Miles promotion should have given her about double the quoted mileage.

In the end, it turns out that the LHR -> BRU leg was not entered onto the system, so the Van Damme’s clerk proceeds to enter this leg which awards Jane an extra 217 miles and says:

“Apologies for the mistake Madam. However, you only have 5947 miles after including the missing leg so you still haven’t got the necessary 10,000 miles required for an upgrade to business class”.

The clerk cannot enter miles directly onto the system for security reasons and he is unable to trigger the promotion retroactively. Jane is told that she would need to contact the Customer Care to look into the issue. After giving up, Jane asks one more question:

“I need to come back to Bangkok in two months time again, if I fly directly from London, would I have enough miles to fly both in and out in business class without the need to come via Brussels in order to double my miles?”

The clerk replies: “That is looking too much ahead into the future. If you don’t get the necessary miles you can always top them up by taking one more flight”. It is a calculation way too complex for the Clerk to perform and the system does not allow to enter hypothetical routes either.

Jane decides to never fly with Van Damme Airlines again.

## OCaml Model

These are a few minor imports that are required by the examples to follow:

```ocaml

```

We first define a few types to describe our problem domain which includes airports, miles, routes and a special type of route called “last route” which may double the points of a following route:

```ocaml

```