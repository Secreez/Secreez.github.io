I just finished presenting our reproducibility infrastructure for a COVID-19 mortality study to a group of master's students. About 16 people showed up smaller than we'd hoped, but shit happens.

One student asked for the GitHub repo link. Another asked how to learn R. Simple questions, but they hit the core issue: **reproducibility isn't about technical perfection. It's about being useful to the next person who touches your work.**

There's no course that teaches this. You just figure it out by thinking about who's going to use what you built and what they'll need when they try.

For our project, I had two people in mind the entire time: the auditor who wants to break the code, and the interested reader who wants to understand it. Every decision you're about to see? Built for one of those two people.

> But before I'm rambling like an LLM and eating your attention span as breakfast...

## **What "Reproducibility" Actually Means**

The [National Academies of Sciences in Reproducibility and Replicability in Science.](https://www.ncbi.nlm.nih.gov/books/NBK547531/) defines it like this (National Academies, 2019):

> **Reproducibility** is obtaining consistent results using the same input data; computational steps, methods, and code; and conditions of analysis.

In other words: someone else takes your data and your code, runs it, and gets the same numbers you did. Bitwise identical, ideally. Or at least within an acceptable range of variation.

This is different from **replicability**, which is:

> Obtaining consistent results across studies aimed at answering the same scientific question, each of which has obtained its own data.

**Translation:**
- **Reproducibility** = same data, same code, same result
- **Replicability** = new data, similar methods, consistent conclusion

For this post, I'm focusing on **reproducibility** - the computational kind. Can someone else run your pipeline and verify your claims? That's the bar we aiming for here.

## **Reproducibility in Practice (The Usual Advice)**

Standard reproducibility advice looks like this:

- Use version control (Git)
- Document your code (comments, README)
- Pin your dependencies (`requirements.txt` in Python, `renv` in R)
- Write a clear methods section
- Share your data (if legally/ethically possible)
- Profit?

**Cool. Technically correct. Completely useless as guidance.**

Because it doesn't tell you *why* those things matter, or *when* they actually help, or *who* they're for. It's prescriptive rules without the underlying logic really.

Here's what actually works: **Think about your stakeholders.** (Big surprise, I know.)

## **The Two-Stakeholder Framework**

...

