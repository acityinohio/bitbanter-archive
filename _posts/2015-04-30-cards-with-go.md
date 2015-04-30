---
layout: post
title: Over-Engineered Solutions to Trivial Probability Problems with Golang and Concurrency
---

*Or how I learned to stop worrying and love brute force and sync.WaitGroup*

A month or two ago, a friend had a 30th birthday celebration in a beautifully rustic cabin on the outskirts of Luray, VA. During frigid February nights, the fireplace and company brought warmth while the snowcapped Appalachians dotted the horizon, like foamy rockwaves frozen in time. It was serene, bucolic, idyllic.

And there was no goddamn Internet.

Which normally wouldn't have been a problem; finding oneself #disconnected is food for the soul. Until something funny happened. We were playing a card game, whereupon each player was dealt 10 cards. During one of these hands, I looked at my cards, and was tickled to find that (ignoring suits) they were in ascending order.

Now what are the chances of that?

Having majored in Math, I had promptly forgotten everything about combinatorics that could attack this problem analytically. But I had a computer, and the complete lack of social graces to ignore company for a few hours while I figured out how to brute force the problem programmatically. And it was a fun excuse to use some Golang concurrency primitives. (and was easy to parallelize as a bonus!)

Before going into the code, let me first explain what you're looking for when you brute-force probability problems...though you might find a better explanation at the [Monte Carlo method](https://en.wikipedia.org/wiki/Monte_Carlo_method) Wikipedia page. What you're looking for is the following ratio:

	number-of-times-event-happens / some-very-large-number-of-attempts

In the case of an ordered hand from a shuffled deck, the larger the hand, the more attempts you need to come up with a reasonable approximation of probability. For 10 cards, it's so rare that you need something on the order of millions and millions attempts to even begin to feel confident about your sample. Too many hands to do in my lifetime.

But not too many for a simulation with my trusty laptop! I'll have a full Gist at the bottom of this post, but let's go through the key parts of the code.

```go
var wg sync.WaitGroup

func probSorted(size int, trials float64) (prob float64) {
	//Make channel to relay whether deck is sorted
	sorted := make(chan bool)
	//Number of times deck shows up true
	truth := float64(0)
```

First, I create a globally-scoped "sync.WaitGroup," which is an incredibly useful type for avoiding deadlock with a hard-to-count number of concurrent functions (I'll explain more about how to use a WaitGroup in a minute). My function takes the "size" of the hand, the number of "trials" you want to perform, and returns the "prob"-ability of the hand being sorted. I make a channel of bool values called "sorted," which will be sent a value when a shuffled hand appears sorted from functions run concurrently later on. And I'm initializing "truth" as the "number-of-times-event-happens" numerator in the expression above.

```go
	for i := 0; i < int(trials); i++ {
		//Increments waitgroup
		wg.Add(1)
		//shuffle a new virtual deck, decrements waitgroup when done
		go shuffleDeck(sorted, size)
	}
```

In the same function, I set up a loop to spin off my trials of shuffled hands (through the "shuffleDeck" function) which are run concurrently, which means the loop doesn't wait for the function to finish before continuing. You'll notice the WaitGroup is being "incremented" here. Before every function is run, a running increment within the WaitGroup is increased by 1...which makes keeping track of our concurrent functions a breeze (and enables us to block the program from finishing until the WaitGroup decrements to zero). We'll wind up manually decrementing the WaitGroup within the "shuffleDeck" function, which is how we reach zero there in the first place.

```go
	//Concurrently waits for all goroutines to finish to close the "sorted" channel
	go func() {
		wg.Wait()
		close(sorted)
	}()
	//Loops through sorted channel incrementing true cases
	//Finishes loop when it receives "close" on the sorted channel
	for range sorted {
		truth++
	}
	//Probability is number of true cases over trials
	prob = truth / trials
	return
}
```

That WaitGroup again! In this part of the function, we're concurrently wait for the WaitGroup to reach zero. Once it does, we close the "sorted" channel, which enables us to block the program in the next for loop until it receives the close signal on the channel. An aside: when you use a Golang "range" on a channel, you need to send a "close" over the channel, otherwise your program won't know when to end the for loop. That's what the WaitGroup is really for...to help us know when to close the channel, so we can finish the program, confident we've done every single trial concurrently.

At the end of the function, we're returning the probability of the event, which is the number of times the shuffled hand appears sorted over the number of trials.

Now the only thing we have to check out is the shuffleDeck function, which you'll find in its entirety below:

```go
//This function copies a new deck of cards, creates a new hand
//of size size, sends "true" down the sorted channel if it's sorted
func shuffleDeck(sorted chan bool, size int) {
	//Decrement waitgroup when finished
	defer wg.Done()
	//Keep two latest cards of trial hand
	var tryhand1, tryhand2 int
	deck := []int{1, 1, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5,
		6, 6, 6, 6, 7, 7, 7, 7, 8, 8, 8, 8, 9, 9, 9, 9, 10, 10, 10, 10,
		11, 11, 11, 11, 12, 12, 12, 12, 13, 13, 13, 13}
	for i := 0; i < size; i++ {
		windex := rand.Intn(52 - i)
		tryhand1, tryhand2 = tryhand2, deck[windex]
		//Remove item from deck
		if windex == 0 {
			deck = deck[1:]
		} else if windex == 52-i {
			deck = deck[:52-i]
		} else {
			deck = append(deck[:windex], deck[windex+1:]...)
		}
		if i > 0 && tryhand2 < tryhand1 {
			return
		}
	}
	//If loop completes without returning, then trial deck is sorted
	sorted <- true
	return
}
```

There's a few "gotchas"/"edgey" things you have to be careful about, but otherwise the function is pretty straightforward. We take the size of the hand, pick a few cards, and if we pick a lower number than the last card, we return the function. Otherwise, if we complete the loop without returning, we send a "true" value down the "sorted" channel, which our other function is counting concurrently.

Notice the super-slick use of "defer wg.Done()". wg.Done() decrements the WaitGroup by one, which we need to know in order to close the "sorted" channel...and Golang has a wonderful "defer" operation that lets you guarantee that an expression is done right before a function returns. Really nice way to organize your code, because you can "defer" things that need to get done at the very beginning of your function, making lots of things more readable. Be careful though: it will evaluate variables in that defer statement wherever your statement appears, creating some nasty bugs if that variable is modified within your function. But it's great for wg.Done().

The full program [can be seen in this Gist](https://gist.github.com/acityinohio/792a1f1b45b7a0fdc841), and you'll notice since there aren't any crazy memory sharing issues, it can be easily parallelized by adding this line to your main function:

```go
//replace "whatever_your_CPU_max_proc_might_be"
runtime.GOMAXPROCS(whatever_your_CPU_max_proc_might_be)
```

So, after all that work, what's the probability that a 10 card hand is shuffled in order? Well, after about two minutes of computation on a Core i7 and with 200,000,000 trials, my computer says: 2.435 * 10 ^ -6. Or a little more than twice for every million hands shuffled.

What's the probability that I won't get invited to the next cabin trip? Uncomputable, but assuredly a great deal higher.
