---
layout: post
title: 5 tips on how to use GitHub Projects effectively
tags: [github,project-management]
comments: false
readtime: true
slug: how-to-use-github-projects-effectively
---

[GitHub Projects](https://docs.github.com/en/issues/trying-out-the-new-projects-experience/about-projects) 
is a recently released GitHub project management tool that provides a flexible way to organize work.
Despite that it is still in beta, I think this is a big step forward.
But, with great flexibility comes great uncertainty.
That is why, in this article, I want to share some tips on how to use GitHub Projects effectively.

## 1. Use draft issues

Draft issues belong to your GitHub Project 
instead of a repository like regular issues.

![Example of a draft issue in GitHub Projects](/assets/posts/github-draft-issue-example.png)

In addition, you can assign fields values to created draft issues (except *"Labels"* and *"Milestones"* fields as they require a regular issue).

![Example of a draft issues in a view in GitHub Projects](/assets/posts/github-projects-draft-issues-view.png)

No need to write tasks in an external place like Google Docs.
You can keep them as draft issues in your GitHub Project.
And with custom fields that represent, for example, priority and size, 
you can sort and filter tasks to understand what matters the most.
The draft issue can be converted to a regular issue with a click of a button later.

*Docs:* 

- [How to add draft issues to your project?](https://docs.github.com/en/issues/trying-out-the-new-projects-experience/quickstart#adding-draft-issues-to-your-project);
- [How to convert draft issue to issue?](https://docs.github.com/en/issues/trying-out-the-new-projects-experience/creating-a-project#converting-draft-issues-to-issues)

*Examples*. Projects that are using draft issues:

- [Pulumi Roadmap](https://github.com/orgs/pulumi/projects/44)
- [Level](https://github.com/orgs/Level/projects/3)

## 2. Use "Iteration" type field

The "Iteration" field type allows you to create flexible time ranges 
within a project and assign them to your issues.

You can see a field named "Sprint" that has the "Iteration" type below.
Time ranges can be different, customized separately.

![Screenshot of possible Iteration field type values](/assets/posts/github-projects-iteration-field.png)

Now, in the project, you can assign created time ranges to a specific issue.

![Assigning Iteration type field to the issues](/assets/posts/github-projects-iteration-field-usage.png)

This field type is useful if you are following agile practices.
You are able to track issues over time and plan work ahead.

In addition, in the same way as with other fields, 
you are able to group and filter time ranges.

*Examples.* Projects that are using "Iteration" type field:
- [Alerta](https://github.com/orgs/alerta/projects/3)
- [Level](https://github.com/orgs/Level/projects/3)


## 3. Use "Linked Pull Requests" field

This field contains a list of pull requests linked to the issues in your project.
No need to leave the board to see what PRs are linked. 
The field is created automatically for each project. 
You only need to include it in your view. 

*Docs:* [How to include/hide fields in the view?](https://docs.github.com/en/issues/trying-out-the-new-projects-experience/customizing-your-project-views#showing-and-hiding-fields)

![Screenshot to show how Linked Pull Requests field looks](/assets/posts/github-projects-linked-pr-field.png)

## 4. Use icons for your field values

Use icons to make field values more recognizable and better looking.
The best way to illustrate the tip is by visual comparison.
Check two screenshots below. What do you think?

*Without icons*
![Not using icons for field values iin GitHub Projects](/assets/posts/github-projects-noicons.png)

*With icons*
![Using icons for field values iin GitHub Projects](/assets/posts/github-projects-with-icons.png)

Aren't values with icons look better? But, how to add them to your field values? 
Follow the steps:

1. Go to *"Settings"* page;
2. Under the *"Fields"* tab, click on the field you want to change;
3. For each field value, add emoji of your choice in a form of `:emoji_name:`,
e.g `Low :star:`.

Below you can find animated step-by-step guide on how to add icons to field values.

![GIF with step-by-step guide how to add icons to field values](/assets/posts/github-projects-icons-guide.gif)

Not sure what emojis exist? 
Check [the list of emojis](https://gist.github.com/rxaviers/7360908) supported by GitHub.

*Examples*. Projects that are using icons for their field values:

- [Pulumi Roadmap](https://github.com/orgs/pulumi/projects/44)
- [Alerta](https://github.com/orgs/alerta/projects/3)
- [Level](https://github.com/orgs/Level/projects/3)

## 5. Follow the development 

If you are excited about new features or waiting for missing ones,
you can do a couple of things:

- check regularly [GitHub Issues Changelog](https://github.blog/changelog/label/issues/) to get the latest updates on new features and improvements;

- check once in a while [GitHub Public Roadmap](https://github.com/orgs/github/projects/4247/views/7) (go to *"GitHub Issues"* view) 
to see what features to expect in the future.

## Conclusion

I hope you find the tips and examples useful.
GitHub Projects is in active development, and things might change a lot.
As time goes, I will update the article with new information.
Thank you for reading!