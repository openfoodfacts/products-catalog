# Software guidelines

Those are some guidelines about the project.

## Always async code

Be very cautious to keep the code asynchronous.
It's important to have good performances and ease scaling of the service.

## Transaction at request boundaries

By default there should be a transaction at request boundaries.
This is the best way to ensure database consistency.

This mechanism, can be escaped for API boundaries that needs it,
but it should be explicit and explained in the code.

Any database update must be in a transaction.

## Comment your code

Here are some rule on how to comment the code:
1. keep things concise
1. put a comment summarizing medium/long block of code,
   so that the reader can easily preview what it does
   (rapid understanding or scrolling to a sweet spot)
1. explain the why and how
   the code is doing something whenever it's not obvious.
1. You may add TODO to document possible enhancements,
   FIXME should be temporary and never committed to main branch


## Testing

Testing is a very important part of this project,
as we want a very reliable service.

As the complexity is more in the integration than in specific logic,
there should be a consequent share of integration tests,
eventually testing some boundaries (eg. race conditions).

Complex functions should come with unit test.