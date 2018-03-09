
---
layout: post
comments: true
title:  "How to do Machine Learning Research"
excerpt: "Or thrive in uncertainty more generally"
date:   2017-8-31 22:50:00
---

<!-- If you're not familiar with research in computer science, it basically consists of the following:

 1. Find a hard and/or important problem involving computation. Examples might include making speech recognition software work better, building a better data compression algorithm, or designing a better programming language.
 2. Spend months trying to figure
 -->

As a PhD student, I spend the vast majority of my waking hours conducting research. <!-- If you're not familiar with computer science research, it basically consists of solving problems involving computing in a manner that's better than anything out there already. Examples might include making speech recognition software work better or designing a new compression algorithm. -->
I'm by no means the world's best researcher, but I've learned a great deal since I started. This post consists of what I wish someone had told me at the outset. Hopefully it will also serve to elicit comments from people much better at research than me.

## Objective


## Use Intensive Laziness

Working harder can get you a factor of 2x over other people, but avoiding unnecessary work can get you 5x or 10x, so it’s almost impossible to overstate the importance of being “lazy”—not just at the macro level of what you spend a day on, but the micro level of avoiding every line of code and paragraph of a paper possible.


## Big ol brainstorm/outline

prioritizing:
    -"intensive laziness"
    -greedy order of certainty that you'll need results
        -try to disprove your ideas with the most tenuous hypothesis/assumptions first

mercilessly ferret out assumptions
    -assume that nothing will work unless you've directly tested it, or seen someone else do so in a convincing way

objective function:
    -paper quality per unit time
        -how many people care about the problem * how much they care * how well you solve it * probability you actually solved it this well (based on results)
    -papers are each about a secret
        -most applied science papers can be summarized as "It turns out that if you X, you get Y"
            -E.g., AlexNet (X, Y) is roughly ("use dropout, GPUs, data augmentation, and ReLU activations", "an image classifier that blows everything else out of the water")
        -In math and pure sciences, often more along the lines of "It turns out that X", where X is some fact about the universe
    -other objectives:
        -learning, resume building, notoriety, real-world impact

picking projects (a checklist)
    -look for tight feedback loops
        -minimal code to test a hypothesis; strive for hypotheses per day, not days per hypothesis
        -should be easy to figure out when something is or isn't working, and why
        -avoid long compute times
    -look for problems where you have unique insight or competence
        -can work out the details empirically, but you want a big secret you can leverage into a paper
        -what you'd ideally like is to learn more secrets along the way and use these as the cores of your next papers
    -look for easy proof that you've succeeded
        -if there are existing techniques to compete with, make sure that they:
            1) have published numbers on benchmark tasks you can directly compare to, or
            2) have good open-source implementations that let you run them yourself, or
            3) are so simple that you can code them yourself
        -if applicable, there should be public datasets with ground truth that you can try your technique on
            -I spent an entire summer just getting and cleaning data for EXTRACT
    -look for projects that don't require too much development time
        -e.g., if you have to code your algorithm in high-performance C/C++, that's much worse than just writing it in Python
    -be completely certain that there's a real problem
        -don't assume existing approaches have particular shortcomings because it seems like they should
            -e.g., I thought DTW would struggle when the endpoints were wrong, but actually this barely bothers it
            -in particular, don't assume that anything is too slow
                -besides, you need to make something at least 10x faster and do so in an interesting way for anyone to care
    -and of course, something you think is interesting and important enough to work on

    -it's not research if people already know it will work; but you want to work on things you already know will work (though probably not all the details)
        -this sounds like a contradiction but it's not
            -the first part said "people" know, but the second part said "you" know
                -the key is to have an information advantage over everyone else

solving problems
    -it's often easier to solve an almost impossible problem than an easy one
        -the former will force you to think about the problem differently and try new approach
        -if you aim this high, failing to hit that target can still get you an awfully good solution
    -you basically get good at the problem and build infrastructure to test various ideas rapidly for the first few months, and actually solve the problem in the final weeks before a deadline
        -early on, you're not going to solve the problem; focus on tightening your feedback loops to increase your rate of progress as quickly as possible

    -the probability of an algorithm gaining adoption is inversely proportional to the number of lines of code it takes to implement it
        -you can mitigate this to some extent by releasing a well-documented library with good wrappers

moving fast:
    -find simplest way to test your hypotheses that you can
        -lower and upper bounds
            -example: thought ANN search with LSH would help; try exact NN as an upper bound that was way easier to code

find your style
    -inspired by Josh Waitzkin's *The Art of Learning*
        -footnote: should really be called "The art of becoming #1 in the world". He's a total beast (world champion in both chess and martial arts by age 25) and I would highly recommend it.
    -I like problems that are so hard it's not clear they're possible

levels of detail:
    -picking area / thesis
    -picking projects
    -executing on a project (month-to-month)
    -executing week-to-week
    -executing hour-to-hour
        -link to study hacks
        -link to another post

other refs:
    -hamming
    -feynman?
    -bell stars
    -profile of that harvard guy with all the nature papers
    -a couple other ones

Misc:
    -it's usually better to spend time making your results better than trying to polish or rebrand mediocre results. I once submitted a paper that had results that were okay, but not good enough for the reviewers. We then spent about a month trying to apply our technique to different data where we thought the results might be better. They weren't, and we missed the deadline for the next conference. What we should have done instead was spend that month making our technique better, not trying to make do with its current performance.
    -Use Python or Julia unless you're a statistician, in which case you should probably use R. Please don't use MATLAB unless you barely know how to code or need to use a MATLAB-only library.
    -Python has a [lot of great tools](my python post) that can dramatically increase your productivity

