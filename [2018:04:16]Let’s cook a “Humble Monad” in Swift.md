# Let’s cook a “Humble Monad” in Swift
> Maybe this is the first post in BS blog that was written in English. This is possibly a little bit challenging for both reader and writer. You may find it difficult at first, but don’t worry. You will get it soon.

## Introduction
Hello there!

It’s L.K again the guy who wrote a post about new features in Swift 4 (maybe a year ago!) now come back and share to you something more interesting about implement some Functional Programming (FP) methodology into our favorite Swift lang!

I really crazy about FP. So I come across with any kind of IT book that talks about FP to get something interesting. Then I met up with “Functional Light Javascript” by Kyle Simpson. (You can buy this [book](https://leanpub.com/fljs) to show your appreciate to the author!)

The main chapters of the book build you some fundamental knowledge about FP and give some tips on how to utilize it better in daily production. It’s apparently a very good book for you to start with FP. But the most thing that give me a lot of fun is the appendix “The Humble Monad”.

Hence I decided to make this “Humble Monad” in Swift version.

### Disclaimer!!!
To follow this post you should have:

* Some basic knowledge about Swift
* Some basic knowledge about FP (Closure, Functor, Applicator, Monad)

If you’re ready => Let’s go!

## Monad definition recall
I will recall most threatening definition about monad:

> A monad is just a monoid in the category of endofunctors.

> How to learn about Monads:
> 
>  1. Get a PhD in computer science.
>  2. Throw it away because you don’t need it for this section

> A monad is a value type, an interface, an object data structure with encapsulated behaviors.

Actually, none of those definitions are particularly useful.

I will make it simpler: `Monad is a data type!`

It holds something like an Int, String, Array, even a Function or … whatever!

A monad is how you organize behavior around a value in a more declarative way.

So you may imaging that monad holds a value and some transforming behavior. That’s It! You’re right!

Next, implement some monads! 

## Maybe Monad
It's very common in FP material to cover well-known monads like Maybe. Actually, the Maybe monad is a particular pairing of two other simpler monads: Just and Nothing.

What is Just monad:

> A basic primitive monad underlying many other monads you will run across is called Just. It's just a simple monadic wrapper for any regular (aka, non-empty) value.

What is Nothing monad:

> Nothing is a monad that holds an empty value.

Put it all together:

> Maybe is a monad that either holds a Just or a Nothing.

![](https://camo.githubusercontent.com/6394872b0982b93780d55c01517ffd19335f22f8/687474703a2f2f616469742e696f2f696d67732f66756e63746f72732f636f6e746578742e706e67)

This can be simply implemented in Swift as the code below:

```swift
enum Maybe<T> {
  case Just(T)
  case Nothing
}
```

Maybe also acts as: Functor, Applicator and of course Monad! So some behaviors will come along with it: fmap, apply and chain.

If you have no idea what the heck is Functor, Applicator… -> please visit this blog post [Swift Functors, Applicatives, and Monads in Pictures](http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/).

These behaviors are implemented as below:

```swift
extension Maybe {
  func fmap<U>(_ f: (T) -> U) -> Maybe<U> {
    switch self {
      case .Just(let x): return .Just(f(x))
      case .Nothing: return .Nothing
    }
  }

  func apply<U>(_ f: Maybe<(T) -> U>) -> Maybe<U> {
    switch f {
      case .Just(let JustF): return self.fmap(JustF)
      case .Nothing: return .Nothing
    }
  }

  func chain<U>(_ f: (T) -> Maybe<U>) -> Maybe<U> {
    switch self {
      case .Just(let x): return (f(x))
      case .Nothing: return .Nothing
    }
  }
}
```

Let’s see how we play with Maybe monad:

```swift
// fmap example

let doubleIncome: (Int) -> Int = { $0 * 2 }

let yourWealth = Maybe.Just(200) // => Just(200)

let workHarder = doubleIncome;

let yourNextYearSalary = yourWealth.fmap(workHarder) // => Just(400)

let yourNextNextYearSalary = yourWealth.fmap(workHarder).fmap(workHarder) // => Just(800)


// apply example

let x10ProfitPrinciple = Maybe.Just({ $0 * 10 }) // => Just(Function ...)

let currentCompanyProfit = Maybe.Just(5) // => Just(5)

let nextYearProfit = currentCompanyProfit.apply(x10ProfitPrinciple) // Just(50)


// chain example

let winLottery: (Int) -> Maybe<Int> = { .Just($0 + 1_000_000) }

let youLuck = yourWealth.chain(winLottery) // => Just(1000200)
```

Ok! Nothing is fancy here. So let’s move to the next main part, the Humble Monad.

## The Humble Monad
Now that we have a little more understanding of Maybe and what it does, now we are going to cook the Maybe+Humble monad. Technically, MaybeHumble(..) is not a monad itself, but a factory function that produces a Maybe monad instance.

Humble is an admittedly contrived data structure wrapper that uses Maybe to track the status of an egoLevel number. Specifically, MaybeHumble(..)-produced monad instances only operate affirmatively if their ego-level value is low enough (less than 42!) to be considered humble; otherwise it's a Nothing() no-op. That should sound a lot like Maybe; it's pretty similar!

Here's the factory function for our Maybe+Humble monad:

```swift
func MaybeHumble(egoLevel: Int) -> Maybe<Int> {
  return !(egoLevel >= 42) ? .Just(egoLevel) : .Nothing
}
```

Let's illustrate some basic usage:

```swift
var bob = MaybeHumble(egoLevel: 45) // => Nothing
var alice = MaybeHumble(egoLevel: 39) // => Just(30)
```

What if Alice wins a big award and is now a bit more proud of herself?

```swift
func winAward(ego: Int) -> Maybe<Int> {
  return MaybeHumble(egoLevel: ego + 3)
}

alice = alice.chain(winAward) // => Nothing
```

The MaybeHumble( 39 + 3 ) call creates a Nothing() monad instance to return back from the chain(..) call, so now Alice doesn't qualify as humble anymore.

Now, let's use a few monads together:

```swift
var bob = MaybeHumble(egoLevel: 41)
var alice = MaybeHumble(egoLevel: 39)

func teamMembers(_ ego1: Int) -> (_ ego2: Int) -> () {
  return { ego2 in
    return print("Our humble team's egos: \(ego1) \(ego2)")
  }
}

bob.apply(alice.fmap(teamMembers)) // => Our humble team's egos: 39 41
```

However, if either or both monads are actually Nothing() instances (because their ego level was too high):

```swift
var frank = MaybeHumble(egoLevel: 45) // => Nothing

bob.apply(frank.fmap(teamMembers)) // => No out put

frank.apply(bob.fmap(teamMembers)) // => No out put
```

teamMembers(..) never gets called (and no message is printed), because frank is a Nothing() instance. That's the power of the Maybe monad, and our MaybeHumble(..) factory allows us to select based on the ego level. Cool!

## Humility
One more example to illustrate the behaviors of our Maybe+Humble data structure:

```swift
func introduction(_: Int) -> () {
  return print("I'm just a learner like you! :)")
}

func egoCharge(_ amount: Int) -> (_ concept: String) -> (_ egoLevel: Int) -> Maybe<Int> {
  return { concept in
    return { egoLevel in
      print("\(amount > 0 ? "Learned" : "Shared") \(concept)")
      return MaybeHumble(egoLevel: egoLevel + amount)
    }
  }
}

let learn = egoCharge(3)

let learner = MaybeHumble(egoLevel: 35)

learner
  .chain(learn("closures"))
  .chain(learn("side effect"))
  .chain(learn("recursion"))
  .chain(learn("map/reduce"))
  .fmap(introduction)
// Learned closures.
// Learned side effects.
// Learned recursion.
// ..nothing else..
```

Unfortunately, the learning process seems to have been cut short. You see, I've found that learning a bunch of stuff without sharing with others inflates your ego too much and is not good for your skills.

Let's try a better approach to learning:

```swift
let share = egoCharge(-2)

learner
  .chain(learn("closures"))
  .chain(share("closures"))
  .chain(learn("side effect"))
  .chain(share("side effect"))
  .chain(learn("recursion"))
  .chain(share("recursion"))
  .chain(learn("map/reduce"))
  .chain(share("map/reduce"))
  .fmap(introduction)
// Learned closures.
// Shared closures.
// Learned side effects.
// Shared side effects.
// Learned recursion.
// Shared recursion.
// Learned map/reduce.
// Shared map/reduce.
// I'm just a learner like you! :)
```

Sharing while you learn. That's the best way to learn more and learn better.

## Conclusion
Obviously, It’s harder to implement these methodology into Swift IMO. I have used chainable method way to implement Maybe monad in this blog post. There is still another way calls stand alone function with infix operator to make Swift code looks more Haskell way!

But the main point is, you may get that from the author, that “Sharing while you learn to learn more and learn better”!

I really got excited and inspired from this message. And I hope you got it too! 

Share with your friend if you like this post <3

## Reference
* [Maybe, Reader and Try monad](https://github.com/orakaro/Swift-monad-Maybe-Reader-and-Try)
* [Functional Light Javascript by Kyle Simpson](https://leanpub.com/fljs)
* [How to cook a Monad with Swift - モナドの作り方](https://qiita.com/ysn551/items/31789b29c7e73faf00f2)
* [Swift Functors, Applicatives, and Monads in Pictures](http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/)