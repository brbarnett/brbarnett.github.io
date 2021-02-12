---
layout: post
title: How to write code like a consultant
tags: consulting development
---

![_config.yml]({{ site.baseurl }}/images/2021-2-12-how-to-write-code-like-a-consultant/ask-listen-learn.png)

As software development continues to grow with the world around us and offers more stable and lucrative opportunities, new developers might wonder how to differentiate themselves in an industry that is fairly homogenous and stereotypical. How do you break the mold and stand out from your peers to find more success and happiness in your career?

<!--more-->

I have been working with development teams for over 10 years, including people of all shapes and sizes. My previous few jobs were in consulting, which means I always had clients who dictated what success looked like. Lately with Phoenix Sports Partners, it is up to our product teams and me to define what that means. The theme I have seen throughout for developers who have stuck out to me has come down to their ability to think critically and be consultative - **they are thoughtful, ask good questions and are good communicators**. They also are detail oriented and have good code hygiene.

This is an example of how a standout developer would take a feature from story-to-production:

1. Read the story title and description (and acceptance criteria if available). Think through what you interpret to be the problem and ask clarifying questions if you find any ambiguities. Remember: you are more than a story-to-code translator - it is up to you to determine the full scope of how to meet the objective of a user story. Make sure you understand what the story is trying to accomplish and why. This will influence your ability to find edge cases.
2. Branch from the latest commit on the `develop` branch (make sure to pull first). This is important because it will ensure there are no other unrelated changes in the branch. I occasionally see unrelated changes which makes it difficult to test code.
3. Write some code, test thoroughly. Make sure you're thinking through edge cases at this point to ensure you don't have to return to this work later when bugs are filed.
4. Push your branch.
5. When it comes time to create the PR, please review your own code before submitting. Sometimes things get missed locally in your commits that need to be tidied up before PR creation (e.g., commented code, commits accidentally included from other work).
6. I am not overly strict about encapsulating your changes to _ONLY_ what's written in the story. However, please help me out by explaining any changes that aren't obviously connected.
7. Any code that is not self documenting needs to have code comments. Any PR changes that are not self evident (e.g., are not a part of the story/AC) need to have source control (GitHub, BitBucket, ADO) comments from you to explain. The goal is that a reviewer should have all the context available in the PR to review your code without needing to ask follow up questions.
8. A PR is meant to represent code that you think is ready for production - it's not a staging area. If it's not ready, it should not be in PR.
9. Test all changes once deployed to the first higher-level environment. Typically for me out of my `develop` branches that is a Dev environment.

Throughout this process, always be thinking through the end user experience, and never hesitate to ask questions. I would so much rather field lots of questions than have to throw away or create rework later because we weren't completely aligned on what you're supposed to be doing.

