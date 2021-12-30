---
layout: post
title: How to use GitHub Actions for testing Xcode project? 
tags: [xcode, github, ios]
comments: false
readtime: true
slug: xcode-github-actions
---

Around 1.5 months ago, I have started learning iOS development using SwiftUI. 
I am building a small app to keep track of my finances. 
As part of my learning, I decided to set up a workflow to run Xcode tests using GitHub Actions. 
And today, I want to share my experience of doing it.

## Let's start easy - running tests locally

One small step at a time they say. 
Let’s start with running our tests on a local machine.
I assume you have already created an Xcode project.
In addition, this article is not about writing unit or UI tests. 
If you want some references, please, check [this article](https://www.raywenderlich.com/21020457-ios-unit-testing-and-ui-testing-tutorial). 
But, by default, your project has some empty tests, so you are good to go.

We will use `xcodebuild` tool to run tests. 
Go to your Xcode project directory and run in terminal the following command:

```bash
$ xcodebuild test -project MyProjectName.xcodeproj \
   -scheme MyProjectName \
   -sdk iphonesimulator \
   -destination 'platform=iOS Simulator,name=iPhone 12,OS=15.0'
```

{: .box-warning}
My Xcode version is **13.0** so it supports iOS 15.0.
If the above command failed, please, check the error message. 
There is a chance you have a different Xcode version installed.
The error message will indicate available iOS versions on your machine.
You will need to modify the `OS` parameter in the `destination` field accordingly.

The command will launch both UI and unit tests. 
If things went well, you should get a similar output.

![Output of xcodebuild tool in terminal](/assets/posts/xcode-tests-local.png)

Good job! Let's move to the next step.

## Putting GitHub Actions into action - first workflow

We know how to run tests locally, it is time to move to GitHub.
I don't want to repeat GitHub documentation, so if you never used Actions,
please, check [Quickstart](https://docs.github.com/en/actions/quickstart) 
and [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
articles from the official page. Those are pretty good.

Continuing our journey, here is a simple example of a workflow.

```yaml
name: tests

on: [push]

jobs:
  run_tests:
    runs-on: macos-11
    steps:
    - uses: actions/checkout@v1
    - name: Select Xcode
      run: sudo xcode-select -switch /Applications/Xcode_13.0.app && /usr/bin/xcodebuild -version
    - name: Run tests
      run: xcodebuild test -scheme MyProjectName -project MyProjectName.xcodeproj -destination 'platform=iOS Simulator,name=iPhone 12,OS=15.0' | xcpretty && exit ${PIPESTATUS[0]}
```

We are using *macos-11* [runner](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#runners) 
which provides us with various Xcode versions.
Then, we select Xcode version (in our case 13.0) 
that allows us to test the app with a desired iOS version (in our case 15.0).

{: .box-note}
A list of supported Xcode and iOS versions for *macos-11* runner is available 
[here](https://github.com/actions/virtual-environments/blob/main/images/macos/macos-11-Readme.md#installed-simulators).
You can find similar lists for other *macos* runners 
[here](https://github.com/actions/virtual-environments/tree/main/images/macos).

In the end, we run the same command to run tests as in the previous step but with one addition. 
We are using [xcpretty](https://github.com/xcpretty/xcpretty#usage) 
to format xcodebuild results for better readability.

## Running job with different iOS versions 

You can use [Action's matrix](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#example-including-new-combinations) feature
to run a job with multiple parameters.
The workflow below will run two jobs: one with iOS version `15.2` and second with version `15.0`.
As before, we also need to specify appropriate Xcode version. 

```yml
name: tests

on: [push]

jobs:
  run_tests:
    runs-on: macos-11
    strategy:
      matrix:
        include:
          - xcode: "13.2"
            ios: "15.2"
          - xcode: "13.1"
            ios: "15.0"
    name: test iOS (${% raw %}{{ matrix.ios }}{% endraw %})
    steps:
    - uses: actions/checkout@v1
    - name: Select Xcode
      run: sudo xcode-select -switch /Applications/Xcode_${{ matrix.xcode }}.app && /usr/bin/xcodebuild -version
    - name: Run unit tests
      run: xcodebuild test -scheme BudgetOnTheGo -project BudgetOnTheGo.xcodeproj -destination 'platform=iOS Simulator,name=iPhone 12,OS=${{ matrix.ios }}' | xcpretty && exit ${PIPESTATUS[0]}
```

I added a parameter called `name` to the job to make it appear better. Results:

![GitHub Actions web interface showing two finished jobs](/assets/posts/xcode-github-actions.png)

You can also use matrix feature to test different device configurations, etc.

## Disabling UI tests

When developing my app, I didn't want to have UI tests
at the beginning. I expect the interface to change a lot.
Even without having to write new UI tests, 
there are still some empty tests that take time to initialize.
This is important because 
GitHub Free plan allows only for 2000 minutes per month. 
And **1** minute of macOS runner is **10** minutes of a regular runner 
([details](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions#included-storage-and-minutes)).
I would like to save some free minutes I have :wink:

Considering all the above, I decided to disable UI tests. How to do it?

You will need to disable them in your [project's schema](https://stackoverflow.com/a/20637892).
Go to Xcode, then *Product > Scheme > Edit Scheme*, uncheck "Enabled" field near the
"MyProjectNameUITests".

![Xcode scheme edit window with disabled UI tests](/assets/posts/xcode-disable-ui-tests.png)

## Conclusion

Congratulations! I hope you have your GitHub Action workflow up and running.
This is a sufficient setup for the beginning, but you might want to go further with your CI.
For example, adding a [linter check](https://github.com/realm/SwiftLint) for Swift, 
or writing some UI tests. 
Thank you for reading this article! See you.
