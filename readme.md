# What is Purr?

Purr is a programming environment that aims to realise a very specific view of "programming". And this view is influenced by many non-conventional fields, such as sociology. As well as conventional ones, such as mathematics.

When people talk about "views of programming" they tend to mean something like models of computation (e.g.: Functional Programming is a view of programs as a mathematical pursuit, in a way that helps reasoning about programs statically), or an approach to programming (e.g.: programming in industry is more interested in practical tools to tackle problems than in mathematical consistency).

In a sense, Purr is closer to a new approach to programming. This approach is guided by the following principle:

> “We should be able to collaborate with computers in the same way we do with humans.”

As a view of programming, Purr has a focus on collaboration. When people talk about collaboration, they generally mean helping humans get work done together. That's a very good start. Purr aims to extend that view of collaboration to include computing systems as well. Purr's approach to programming is focused on making practical the process of collaboration among all of these different actors, computers and humans alike. 

Collaboration involves two very important principles:

  - Rich interaction and feedback; and
  - Trust;

We'll discuss this in more details in the following sections.


## Rich interaction and feedback

Programming language design seems, largely, to be driven through the necessities in computing systems and mathematical models. Now, this isn't a bad thing. Those are certainly two things that every reasonable programming language *should* be considering in their design. But programming is as much about humans as it is about those.

The human aspect of programming language design seems to be often neglected. There are very few empirical studies on the usability of programming languages, whereas we have no shortage of studies proving mathematical properties about the latest features (these days, probably gradual typing and effect systems).

Purr proposes that not only we take human usability into account, but we look at systems of people as an inspiration for programming languages, too. In particular, we look at how people work together both when physically in the same location, and remote. Collaboration is shaped by the available tools, and the collaboration enabled by being physically in the same location is still much richer today.

How exactly is this collaboration "rich", though?

Humans use a myriad of forms of communication and medium depending on what the context asks for. We use different pitch and tone of voice to convey information and emphasis. We get physical with the things we're manipulating. We present the same information in different formats. When someone doesn't understand something, we can work *with* them to build an understanding. 

Meanwhile, programming computers is more akin to writing letters. And to someone who'll yell "NO" at you every time your writing differs from the normative grammar. Or when you write something confusing or ambiguous. It's amazing that we have these very rich display devices, touch inputs, and even augmented reality with motion sensors... but the state-of-the-art of programming is still typing and looking at text in 70's terminal emulators.

We can try making computer programming more akin to how we interact with the real world around us:

  - By having rich, immediate feedback;
  - By allowing multiple ways of interacting with the program (e.g.: direct manipulation);
  - By making the programming process an interactive conversation between humans and computers;
  - By supporting multiple users to interact with the same world at the same time;


## Trust in computing

When we talk to people in our daily lives, we don't place the same amount of trust in them. The amount of information we choose to share (and how we share it) differs greatly from when we're talking with a close friend to when we're talking to a work acquaintance. Yet, in programming, we're generally forced to place the same levels of trust in everyone.

Sometimes, we're allowed to manage trust in programming through identity. However, such identity is not very fine grained. We can't (usually) manage different trust levels when talking to Alissa at work and talking to Alissa while hanging out at her place in the weekend (because she invited you to a bbq her wife decided to throw out of the blue). But, worse, we (usually) can't even manage different trust levels when talking to Alissa and when talking to Max. We have to place them in some arbitrary category (say "friends"), and pretend the experience, context, trust, relationship, etc. are the same for all of them, all the time.

Of course, trust is important for security (and this is where you'll usually see this being discussed). But more than that, trust is a *fundamental part of human interactions*. And we should be able to extend them to computer systems as well, at every part of the process.

We can try accounting for trust in the process of interacting with computer systems by:

  - Being able to assign different trust levels to each component and actor in the system;
  - Making trust boundaries and actions visible/observable in the system;
  - Making it easier to understand risks, tradeoffs, etc. when considering whether to trust something/someone or not;
  - Making it easy to take back our trust on something/someone when things go wrong;


## References

  - [An Empirical Investigation into Programming Language Syntax](https://dl.acm.org/citation.cfm?id=2534973)  
    — Andreas Stefik, and Susanna Siebert (2013)

  - [Robust Composition: Towards a Unified Approach to Access Control and Concurrency Control](http://www.erights.org/talks/thesis/markm-thesis.pdf)
    — Mark Samuel Miller (2006)

  - [Programming as an Experience: The Inspiration for Self](http://bibliography.selflanguage.org/programming-as-experience.html)
    — Randall B. Smith, and David Ungar (1995)

  - [The Early History of Smalltalk](http://worrydream.com/EarlyHistoryOfSmalltalk/)
    — Alan C. Kay (1993)