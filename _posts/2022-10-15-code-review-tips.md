---
layout: post
title: Code Review Tips
tags: []
comments: false
readtime: true
slug: code-review-tips
canonical: https://medium.com/@gdscbudapest/code-review-tips-ac3dea32312b
---

Code reviews are standard practice in most software companies. It helps to improve code correctness and quality and catch bugs early on. Because you will spend a lot of your software development time on code reviews, being good at it is essential. The code review involves two parties: *the reviewer* and *the contributor*. The most common code review process looks like this:

1. Contributor makes changes to the codebase (probably following [Github flow](https://docs.github.com/en/get-started/quickstart/github-flow) or similar);
2. After changes are finalized, the contributor opens a [Pull Request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews) (in Github) or [Merge Request](https://docs.gitlab.com/ee/user/project/merge_requests/reviews/) (in Gitlab) or similar request in another code tracking system, with intentions to get their changes added;
3. Contributor requests a review from a reviewer;
4. Reviewer will check code changes and provide comments;
5. After all comments are resolved, the changes will be merged.

An example of Github Pull Request (PR):

![Screenshot of dummy GitHub PR](/assets/posts/code-review-tips/dummy_PR.png)

The process is usually asynchronous, meaning the contributor and reviewer are not present simultaneously and will exchange comments/messages online to communicate.

What makes a good code review? I want to share tips for improving your code review process in this article. The code review tips will be mostly about efficient communication between contributor and reviewer. Let's get started.

{: .box-note}
The article was created for Google Developer Student Club at the Budapest University of Technology and Economics. Please [read the original version](https://medium.com/@gdscbudapest/code-review-tips-ac3dea32312b) and check [their blog](https://medium.com/@gdscbudapest) for more informative articles!

## Tips for both contributors and reviewers

Let's start with a common tips that are applicable for both sides.

### Be respectful

Code review is not a fight. It is a collaboration between people that *hopefully* want to achieve the same goal - contribute the best version of the code they can with the time and resources at their disposal. Consider the other person(s) to be on your side. If you have personal issues with someone, address them outside of the code review process.

In addition, remember, in code review, you criticize the code, not the person who wrote it. So, it is also important not to treat code criticism as a personal attack.

### Code review is an opportunity to share knowledge

Usually, code reviews are mainly viewed as a way to test and ensure the code's correctness. But what is crucial and often forgotten is that code reviews are a fantastic way of sharing knowledge!

Whether contributing or reviewing the changes, look for opportunities to share knowledge. It can be by giving more details about the bug you discovered, explaining the improvement you suggested, or providing a context specific to the team.

Knowledge sharing brings a lot of benefits:

- speed up mentoring for junior engineers;
- onboard new engineers faster;
- or reduce gaps in knowledge among the team members.

### Give praises

An appreciation is a great way to make the code review atmosphere nicer and show how much you value the team's effort. So, in addition, to looking for issues in the code, don't forget to leave praises along the way. Of course, make sure the praises are valid. Don't use praise just for the sake of it. I am sure you can find small and big things worth appreciating in every code review.

### Use Conventional Comments

Conventional Comments is a system to label comments with prefixes that allow communicating intentions more clearly. Setting clear intentions for the comments will help you avoid some misunderstandings. For example, help distinguish between nitpick comments (something trivial, e.g., typos) and more in-depth ones (e.g., potential bugs or a clarification question). You can check [the official website](https://conventionalcomments.org) to find more details and examples about the system.

## Tips for contributors

 It is important to remember that the reviewer is probably not familiar with the changes you made. They will need to dig into your code that might be obvious to you but not to them. Your job as a contributor is to help them review as smoothly as possible. Let's look into some advice below.

### Provide context to the change you are making

When you are working on implementing some feature or bug fix, it might not be evident to the reviewer why you choose one approach or the other. Adding a bit of context and history to describe your change is always good.

For example:


> This pull request fixes bug discovered in #123 issue. Initially, I tried to fix it directly doing [describe your steps] but it didn't work out because of A. So, I ended up fixing the issue like this B.

The context will provide necessary information for the reviewer and help them to avoid wasting time re-discovering things.

### Add "How to test?" section

The "How to test?" section describes how to run the code changes step-by-step. This tip is in the same area as before - provide more context to make the reviewer's life easier.

It will help the reviewer start testing changes as fast as possible, allowing them to focus on understanding your code instead of figuring out how to run it.

For example, such section can look like this:


> How to test this pull request?
>
> 1. Add new rows to the "Product" table in your local database using command: [put exact SQL/bash commands here]
>
> 2. Run previous version of the code using [put exact command here]. You can see that rows are not updated as expected
>
> 3. Now run code with the changes. You can see that price is properly updated for all rows. The issue should be fixed
>
> 4. I have tested that prices are correctly re-calculated but maybe checking [describe case] also makes sense. 
> I could miss something. Unit tests are covering a few cases, maybe, I can add more edge cases there. What do you think?

### Follow contributing guidelines

The contributing guidelines usually contain code style, licensing, and other relevant information needed to add your changes. Open-source projects are known to have and follow [the guidelines](https://en.wikipedia.org/wiki/Contributing_guidelines).

If your code changes are not compliant with the guidelines, they will be rejected, or you will probably be asked to adjust the changes. So, check and follow contributing guidelines used by your team or project before requesting a review. This will save time for both you and the reviewer.

## Tips for reviewers

As a reviewer, you must understand that you, like the contributor, are also responsible for the code changes. Do not treat the review process as some unpleasant activity. Help contributor to finalize changes. Let's see some of the advice.

### Fetch, run, and test the code

The software can quickly become highly complex. So even if automated tests pass, it doesn't mean that code will always work correctly. That is why it is still important to run the code changes by yourself, debug it, try to break.

This is where the "How to test?" section comes in handy. It will encourage and help the reviewer to run the code.

### Provide details to your suggestions/comments

What code suggestion looks more informative and will help contributor?

Comment number 1:

> This function has a long body. Try to split it up into smaller functions.

Or, comment number 2:

> This function has a long body. It is fetching, filtering and saving data in the same body. Let's extract filtering and saving into separate functions. Later, we will be able to re-use them as we filter and save data in the same way in other parts of the code. You can put a new functions to file A where we keep all common code.

Comment number 1 will leave contributor guessing what "split it into smaller functions" mean. Comment number 2 is much useful, explains reasoning and doesn't leave much room for contributor to second guess the meaning. If I were the contributor, I would definetely prefer to receive a comment number 2.

In case you "smell" that something is wrong, but not sure how to improve the part you can still provide a more descriptive comment like:

> This function has a long body. I spent time thinking how to split it into smaller functions but couldn't come up with anything reasonable. Do you have some thoughts?

This advice plays nicely with the one before - sharing the knowledge. This is example of how it may look like:

![Screenshot of code change in GitHub PR](/assets/posts/code-review-tips/list_example.png)

The comment number two definitely looks more informative and supportive.

## Conclusion

To summarize this article, I would say that good writing is vital during code review, and learning to communicate coherently is essential. Good writing skills become even more critical when you have a remote job where people are not present simultaneously; hence it is harder to clarify things on the spot.

Hopefully, this article gave some valuable tips about better code reviews.

