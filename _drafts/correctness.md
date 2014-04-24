Program correctness
-------------------

Correctness issues may seem off-topic for this article, which is declared to be about optimisation, not about testing. However, maintaining program
correctness is one of the key points of the optimisation process. Often programmers (and even more often, managers) are scared of any optimisations,
considering the risk too high. This approach is summarised in a well-known engineering rule: "Do not fix what is not broken".  This means that if the
optimisation process isn't preserving the program correctness, the managers will never allow programmers to embark on it, which will make our life
dull and boring forever. That's why I have to say a few words on the issue.

The performance improvement process starts at a point where there is already a solution that works - or, to be pedantic, that we have reasons to believe works.
Such a belief may originate from code inspection, ideally performed by several people other than the author of the code. It may originate from formal
correctness proof (unfortunately, this is very rare, as correctness proving tools are not well developed yet). And, finally, it may originate from extensive
testing. This is usually what "the program works" means: it has been tested. Program testing is a science on its own, and way too big to be covered in this
article. Proper testing must include verifying program behaviour on valid and invalid data of various sizes and contents, running it standalone and as part
of the system, and a lot more - so we'll simply assume that all this has already been done for the solution we start at. We also expect that it will be done
on whatever better solution we develop.

However, this doesn't mean that we can ignore testing issues completely as we carry on with our improvements. We are not going to run full tests for each
version we create, but it is always a good idea to run a basic check that at least on some realistic input the new version produces the same result as the
original version. This is called regression testing, and the original version is called the reference version, since it serves as the reference point for
comparison. If regression testing is not done, we can end up comparing the performance of a program that works with that of a program that doesn't.
Such comparisons can only produce meaningless results. That's why regression testing must be considered integral part of the performance improvement process.

Regression testing requires a wise choice of input data. It must as much as possible prevent the accidental producing of a valid result by an incorrect program.
For instance, an input of all zeroes is very bad in our case, because it produces the output of all zeroes, which can be easily produced by some incorrect
solution. Filling the input buffer with consecutive byte values (0, 1, 2, .., 255, 0, 1, 2, ...) is better, but it is also vulnerable - imagine a program that
uses half of the input buffer twice. The best option is to use random data (but even that isn't bullet-proof).

Although testing is a generally accepted absolute criterion of correctness, I still wouldn't underestimate the value of code inspection.
It happened to me several times that a bug was found by looking at a code rather than by testing. The code inspection is very useful for
all the versions of code we develop, but it is especially important for the reference implementation. That's why it is crucial that the
reference implementation is written as simply as possible. If there are mathematical formulae or other formal ways to define results,
they must be followed strictly, without any optimisation attempts. I would even advise to go as far as employing two people for developing a piece of code,
exactly one of whom being a performance freak, and the reference implementation being made by the other one.

