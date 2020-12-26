# Maybe Not Reflections
Back in 2018, Rich Hickey's [Maybe Not](https://www.youtube.com/watch?v=YR5WdGrpoug) sparked debate in the team I was working in at that time. It caused us to re-evaluate many of the positions we had previously taking to both types and API design. It also clarified an underlying issue that we had been battling for a long time, namely which type theoretic constructs are appropriate to use across API boundaries to minimize coupling.

A while later I began seeing many comments on Hacker News and Reddit that were critical of Rich's talk. These critiques claimed Rich does not understand (or even seems to hate) type systems and Haskell. I think critique and debate is great, but these comments all seemed to miss the underlying point. It's understandable that they missed the point, I had to personally rewatch the talk several types to really begin to understand it. `Maybe Not` is not an attack on Haskell. It is not meant to deride type systems. This is a talk about minimizing coupling and enabling compatible change. 

## Compatible Changes
Software is almost never written in isolation. The functions you write will end up being used by some other programmer. That other programmer could be you in the future, someone on your team, or someone on the other side of the world that you will never meet or even talk to. Software also changes. That function you wrote will hopefully continously become better over time. Maybe it will be able to handle more use cases or provide stronger guarantees. If the change you makes breaks existing callers of that function, either by forcing them to rebuild, to adjust their types/names, or by causing issues that manifest at runtime, this is called a breaking change. If the change does not break existing callers, it is called a compatible change.

Breaking change is antithetical to software development[1]. It breeds distrust, slows down development, and erases a lot of previously made progress. I remember once buying a 2 year old book about a language, and the code in it was already incompatible with the newer version of that language that I had installed locally. I tried to downgrade the language, but that broke all the tooling I had locally, and once that had been fixed it turned out some transitive dependencies were also causing issues. After several hours of frustration I gave up on running that code. All of this was with a book that was only 2 years old. In that short period of time the ecosystem had changed so much that the book and its accompanying source code seemed like little more than a paperweight. This kept happening with every upgrade of the language and with most library updates. It completely killed my productivity and motivation.

Compatible change, on the other hand, enables software development. Updates to the language and libraries you use should make your life simpler and better, not complicated and frustrating. Updates should be made without fear and ideally completely automatically. A great library from 20 years ago can still be a great library, and if it solves your problem you should be able to pull it into your project and just use it. Software development is fundamentally distributed. Not everyone can update their code all at once. Despite this reality, our community continues to forgive and even embrace breaking change.

The languages we use often make this problem even worse. Changes that logically/semantically should be compatible are often breaking changes in reality simply because of constraints imposed by our programming languages. The `Maybe` and `Either` types (as defined in Haskell) are just two examples of this. These types are meant to encode uncertainty about problems we are dealing with. Over time, we often become more certain, so it would be nice if we could reflect this certainty through compatible changes to our software. However, as Rich demonstrates, `Maybe` and `Either`'s requirement to lift their values into a tagged union mean that such a change is necessarily a breaking change.

Rich also specifically points out that this is not an argument against types in general and he shows how Kotlin's optional types solve this problem in this specific case, and how union types (in langauges like Dotty or Typescript) can solve it generally. This insight about union types is probably the most useful takeaway of the talk for me. I've since applied that style of thinking in large systems I have architected and it makes them substantially easier to maintain and evolve over time. 

To the aforementioned critics: these are simply the reasons that Clojure has not embraced `Maybe` and `Either` to solve the issue of expressing uncertainty. This does not mean that these types are bad, or that Haskell or type systems are bad. These types certainly have some advantages which Haskell takes great advantage of.  Similarly, it does not mean that expressing optionality is not important. Clojure, however, is not aimed at the same types of problems as Haskell is. They simply embrace different philosophies. I think it's wonderful that we live in a world where we can use and learn from both.

## Digging into Optionality
After this, Rich points out that optional types (as expressed in the languages we have today including Clojure), suffer from another problem. That problem is that they are too broad of a solution, so while they may fix a problem in one place, they often spill over to create problems that permeate our systems. There are two questions that always come to mind for me when seeing an optional type: **Why?** and **When?**.

**Why** is it optional? Was it added after the initial structure was created, so existing entities may not have it defined? Is is optional for users of the system to fill out? Is it optional because it is only available at certain points in time, or when certain pre-conditions are met? etc.

**When** can I use this optional field? When should I be setting a value and when should I set the value to `Nothing`? When is the value present, and when should I be especially careful about handling both states? etc.

In reality, we might know the answers to these questions. Maybe the answers are written on a napkin somwhere, or they exist in the head of one of our colleagues. If we are lucky, someone left a comment answering these questions, and if we are extraordinarily lucky that comment is both correct and helpful. The types, though, don't help us much. The constant checking when unwrapping these optional types is not very helpful either. They result in large amounts of extra code, and often the fallback case just involves crashing the program entirely.

These problems really start adding up when aggregates of values are considered. In lists and sets this is typically easily solvable. We just filter out the `Nothing` or `nil` values to turn a `[Maybe a]` into `[a]` in Haskell or `List<Option<A>>` into `List<A>` in other languages. But in tagged aggregates (e.g. records in Haskell, structs in C-like languages, or data classes in Kotlin) we often cannot do this. Type systems in most languages require an entirely new type to be created for each combination of known and unknown values [2]. This explosion of types and names is simply not sustainable, so the aggregates with optional members tend to stick around.

## Taking Things Apart
Rich's proposed solution is to take things apart. He separates the definition of the *schema*, i.e. the shape of an aggregate, from *selection*, i.e. the context in which the schemas are used. For example, an aggregate for a car might specify that the make and model of the car are optional, but a specific function operating on cars should be able to say that it requires a car where the make is known. This separation helps solve many of the questions listed above, especially the ones in the **when?** category. It also helps remove the boilerplate associated with checking by moving that into the rule enforcement system itself.

The final part of the talk shows that this separation actually solves a more general problem - it enables the shape of our data to be decoupled from the usage of that data. I can't begin to count the number of discussions I've been dragged into about how data should be shaped, or how much time I've spent thinking about it and optimizing for it in my own code. These decisions almost always need to be made upfront, and are as a consequence very often wrong. By separating schema and selection, it should be possible to compatibly refactor data into a better shape at a future point in time. This concept still needs to be proven, but I think that if it works it will be something developers will be begging for in their languages in 5 to 10 years from now.

## Conclusion

'Maybe Not' is not an attack on Haskell or type systems; It is an explanation of why Clojure is taking an unconventional approach to optionality, and an exploration of what consequences that might have. Clojure's approach is not necessarily correct, and Haskell's approach is not necessarily bad. They made their own sets of trade-offs, and as programmers we should learn from both so we can choose the right tool for the right job.


[1] To understand Rich's position on this subject, I recommend watching his [Spec-ulation Keynote](https://www.youtube.com/watch?v=oyLBGkS5ICk). 
[2] Typescript's [Mapped Types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types) are the best solution to this that I am aware of in typed language.
