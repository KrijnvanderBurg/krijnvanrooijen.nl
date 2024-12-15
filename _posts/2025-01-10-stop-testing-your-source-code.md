---
title: "[DRAFT] Stop testing your source code, start testing your published package."
date: 2024-01-10
excerpt: ""
tags:
- python
- poetry
- pip
# image: /assets/graphics/2024-12-17-automatic-tests-code-coverage/thumbnail-code-coverage-veiled.png
pin: true
---


## The Fata Morgana of Source Code vs. Packaged Application

When developing software, you typically install dependencies in a controlled development environment, with your source code readily available. Tools like pip or Poetry resolve and install the dependencies, and you proceed with testing. But is this workflow an accurate reflection of how your users experience your application?

Consider the user's perspective: they download a packaged version of your application—often a prebuilt artifact like a .whl file—from a package manager. Their installation process differs fundamentally from yours. Who’s to say that the two approaches—installing dependencies in a source environment versus using a packaged application—result in the exact same application?

Even if the dependency versions match, there’s an unspoken assumption that what works in your development environment will work once the application is packaged and installed. But have you tested this assumption? Did you run your tests on the packaged version of your application, or only against the source code? Packaging itself is a transformative process, and subtle changes can creep in during this stage—whether due to missing files, misconfigurations, or other errors. Why do we assume the result will always be identical? It probably is, but "probably" is not good enough.

Depending on your application and its users, this gap between source and package can have significant consequences. Imagine a user upgrading to a new version of your package, only to find it broken in ways your tests didn’t catch—despite every indicator being green in your development pipeline. This is a failure not just of testing, but of methodology.


## A Practical Shift: Testing Packaged Applications Locally

To bridge this gap, I’ve adjusted my workflow to better reflect the user experience. Instead of installing dependencies and testing directly against source code, I package my application (in this case, for Python) into a .whl file locally. I then install this wheel using pip and run my tests against the installed package. This approach mirrors the way your users will interact with your application, allowing you to identify issues that might otherwise be overlooked.

But this isn’t a practical workflow for everyday development. You can’t expect developers to constantly package and install their application manually every time they want to run tests. This is where automation comes into play. In a previous article, I discussed how to automate your local development environment to reduce manual commands. I have built upon this automation and have mimiced my user experience, effortlessly.

