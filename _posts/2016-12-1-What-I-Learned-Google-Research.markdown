---
layout: post
comments: true
title:  "What I Learned Doing Deep Learning Research at Google"
excerpt: "There's a sense that you're at the center of the AI world"
date:   2016-12-1 22:54:00
---

In summer 2016, I was fortunate enough to intern at Google Research in Mountain View. I learned an enormous amount from this experience, and thought I would share some of it with others. Note that everything here is just my own opinion and/or reflects information that Google has made public.

## People

Google Research feels like a modern day Bell Labs. In addition to legends such as [Jeff Dean](http://www.businessinsider.com/astounding-facts-about-googles-most-badass-engineer-jeff-dean-2012-1), [Sanjay Ghemawat](https://research.google.com/pubs/SanjayGhemawat.html), [Geoff Hinton](https://en.wikipedia.org/wiki/Geoffrey_Hinton), and others, almost everyone is extremely impressive in some way. One guy finished his undergrad, MS, and PhD in 6 years. Another was a chess grandmaster. Another won multiple "best paper" awards in grad school. Another was the #1 student in the entire country of Turkey. One guy is a Thiel fellow and basically [the reason anyone understands neural networks](http://colah.github.io). It felt a lot like EECS at MIT in that you look around and just kind of wonder how you're there.


## Culture and Atmosphere

My experience of Google culture is consistent with what you've probably heard elsewhere. There's unlimited free food, gyms and laundry in various buildings, and a strong expectation that everyone be positive and collaborative. I didn't encounter any jerks, and people always treated one another with respect.

The only aspect I was surprised by is that almost everyone works normal hours, to the point that most of the Google cafeterias aren't even open on weekends. I know people work at home to some extent, but I still wasn't expecting this overall.

At Google Research specifically, there's a sense that you're at the center of the AI world and are inventing the future. Papers that Google writes and systems that Google builds get press coverage around the world. TensorFlow is used by a huge number of companies, researchers, and university classes. If you're in the machine learning world, there are state-of-the-art results, classic papers, and blog posts you know about that were all written by people down the hall.


## Ain't No Problem Like a Google Problem

One of my main takeaways was that Google has *very* different machine learning problems than most other organizations and basically everyone in academia. Below are some of the most important differences.

This section is mostly intended for people with at least cursory knowledge of machine learning, but might be interesting even without it.

#### Data size and sparsity

If you've ever done machine learning on your own laptop or desktop, you probably used a dataset that was a few gigabytes at most, had on the order of hundreds of features, and was mostly dense. At Google, most datasets are **terabytes or petabytes**, have thousands or **millions of features**, and are **extremely sparse**.

Much of this is because Google's core datasets track queries, impressions (i.e., what users saw), and clicks. There are a huge number of queries users have entered and millions of possible videos, web pages, and ads they might have been shown. Each one of these is a unique feature when you [one-hot encode](https://www.quora.com/What-is-one-hot-encoding-and-when-is-it-used-in-data-science) them, and all but a handful of these features are 0 for a given example, since the user only used a handful of search terms and saw a handful of links.

#### Huge number of classes

If you look at datasets outside web companies, most of them have a few thousand classes at most. Google commonly has **millions of classes**, with each corresponding to, e.g., one ad, one youtube video, or one app in the Play store. This means that even a simple linear model can contain an enormous number of parameters. Consequently, there's strong incentive to make models sparse, at least at the output layer. If the model isn't sparse, it can be too slow to serve predictions for live traffic.

Another consequence is that the pattern of retrieval + reranking is common. Retrieval refers to creating a short list of good candidates based on keywords or other simple criteria, while reranking is determining (often with a learned model) which of these candidates are the best.

#### Non-stationarity

Another big difference in the data is that the distribution of Google's data is [non-stationary](https://en.wikipedia.org/wiki/Stationary_process). I.e., what people are watching, searching for, and clicking on changes over time. This is not a problem you ever encounter when using a fixed dataset, which is what almost everyone in academia uses. This non-stationarity also results in decreasing marginal returns from using older data---at some point, the decrease in variance from using more data is outweighed by the increase in bias towards outdated patterns.

#### Data munging

The vast majority of the code in a real-world machine learning system has nothing to do with machine learning algorithms. Instead, it loads data from various sources, cleans it, extracts features, and packages the model(s) for live traffic and live experiments.

#### Every ounce of accuracy matters

When your revenue is directly proportional to the number of clicks or downloads you get, 1% better recommendations mean a 1% increase in revenue. In Google's case, this can mean tens of millions of dollars or more.

#### Bugs are devastating

As a corollary to the above, a bug that causes a model to serve bad recommendations, even for a few hours, can cost millions of dollars.

#### Interpretability matters

Another corollary is that interpretability is often important. Specifically, if Google is going to engineer features and other aspects of its models to get the best possible predictions, it needs to understand why current models are making mistakes.

#### Training and serving costs

When I train a model on my laptop, the only cost I really think about is the time it takes. When you're training on datasets as large as Google's, however, training a model can cost a lot of money. A major result of this is that Google cares a lot about model initialization. In particular, it's desirable to recycle knowledge from previous models to accelerate the training of new models. Training and serving costs are also some of what motivates custom hardware like the [TPU](https://en.wikipedia.org/wiki/Tensor_processing_unit).

#### Feedback loops

Because Google's models affect what users see, there are often feedback loops that make it difficult to infer what the best recommendations are. For example, if my current model recommends YouTube video X whenever you view video Y, this will result in many people viewing X after Y. When I train a new model, it will see this pattern and conclude that people who like Y like X as well. However, it could be that people only clicked on X because that was what we recommended, and there's another video Z that they would like better.

<!-- [DistBelief](https://research.google.com/archive/large_deep_networks_nips2012.html) -->


## Conclusion

Google Research is one of the centers of the AI universe right now, as well as a great place to work. It's an organization I feel incredibly privileged to have been a part of, and I hope this post serves to channel this privilege into something beneficial for those who haven't had the same good luck that I had.
