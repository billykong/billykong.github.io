---
layout: post
title:  "Remote Team Git-based Communication"
date:   2020-05-22 13:15:00 +0800
categories: remote-team
comments: true
mermaid: true
published: true
---

### Motivation
The COVID-19 pandemic has brought many changes to our lives, one of them being many people are working from home now. It is not necessarily a hinterance of communication, but we do need to adapt. Many things that used to be communicated verbally, needs to be communicated textually.
  
We have switched to remote working since January 2020. During the time, we have natually developed a git-based communication workflow. If you are working in the software industry, you will see that it is not unlike the pull request workflow most of us used for source code. The precise and asynchronous communication model for source code works just as well for software design, schedule planning, and team administration discussion.


So far the efficiency is even better than before COVID-19 in addition of better transparency. I would like to share our approach here with some details. For anyone who may be looking for a way to build and run a remote team, I hope this is useful.


### Overview

We call our way **document-based communication**. It is inspired by the handbook-based communication shared by [GitLab][gitlab]{:target="_blank"}. Since our team is much smaller, 4 developers only, our handbook is much simpler.  
  
The handbook is a git repository of markdown files organised into directories. Really, not much difference with your local drive or share drive. But using `git` helps to iterate ideas much faster. Here is the overall directory strcuture:  

```
.
|--engineering-team
|  |--README.md
|  |--merge-request-flow.md
|  `--recruitment-process.md
|--features
|  |--README.md
|  |--feature-1.md
|  `--feature-2.md
|--technical-documentation
|  |--README.md
|  |--investigations
|  |  |--2020-04-01-dummy-issue-1-investigation-report.md
|  |  `--2020-05-20-dummy-issue-2-investigation-report.md
|  |--architecture-diagram.md
|  `--analytics-etl.md
`--README.md
```


### What kind of document is needed for the day-to-day operations?
- Team SOP guidelines
    + This directory holds the day-to-day workflow descriptions.
    + E.g. merge-request-flow.md
        * Simple description of how to do merge request/pull request since not all teammates may be familiar with open source software workflow
        * Define who should be assigned reviewers and criteria for a review to pass
        * Define who is responsible for merging and what to do it there is code conflict
    + Once written, contents in this directory rarely changes.
- Features
    + This directory holds all the new features or features updates requirements
    + If your team has the wonderful practice of writing design doc, here is where it should be kept.
    + For our case, we put the followings guidelines in `/features/README.md`:
        * What should be included in a feature doc?
            - textual description of user flow with links to graphic assets
            - technical design description
            - contentious points
            - estimation
    + If your features is complicated, feel free to open subfolders inside `features/`
- Technical notes
    + This directory is for long-term documentations and technical knowledge sharing, a bit like a wiki or knowledge base.
    + When our teammates consider certain new tech to be added to our services, we require each other to write down what we have: read, opinions, and experience on trying out things here, such that we can learn from each other more easily, and we can monitor the quality of "research" tasks so management can be more comfortable assigning such tasks.
    + The output is very similar to blogs such as this one.
    + We have an `investigation` subfolder here as well
        * When there are tricky bugs or issues, we write down what happened and how we solve them.
    



### Why use `markdown`?
- Easy to write 
    + It takes 30sec to learn for generic document like this one, seriously.
- Easy for source code control
    + It makes the merge/pull request flow possible as elaborated [below](#how-to-communicate).
- This blog is written in markdown, you can find the source [here](https://github.com/billykong/billykong.github.io).

### How to communicate?    
We use the [merge request flow](https://docs.gitlab.com/ee/user/project/merge_requests/). We didn't invent this workflow, it is the common practice used in open source softwares. If you are using [GitHub][github] it is the [pull request flow](https://guides.github.com/introduction/flow/). The similar flow is called merge request flow in [GitLab][gitlab].

Bascially, you do the followings:
1. Create new branch
2. Make changes
3. `git push`
4. Create merge request in gitlab.com
5. Assign reviewer(i.e. Assignee)
6. Done! The reviewer will be notified and be able to comment on your request.

<div class="mermaid">
graph TD
    A(New Requirements) --> B[Create new branch]
    B --> C{File Exist?}
    C -->|Yes| D[Create new markdown file]
    C -->|No| E[Edit exiting markdown file]
    D --> F[git push]
    E --> F
    F --> G[Create merge request]
    G --> H[Assign reviewer]
    H --> J{All issues resolved?}
    J --> |No| K[Edit file]
    K --> |Commit and push| J
    J --> |Yes| L[Merge branch]
    L --> M[New Requirements Communication Successful]
</div>

### What is gained?

1. Efficiency: for our team of 4, we have about 10 threads of communication per day, each is precisely scoped to a line in a document.
2. Constructive disagreement: as teammates has more time to contemplate each comment before dashing it out. Here [emoji][emoji] helps.
3. Sunshine factor: as all comments are recorded and accessible to all teammates. It really encourage the best of behaviour from our experience.
4. Async communication: teammates can check their notification periodically and continue discussion without losing context. And the developers are not interrupted neither! Big win here!




### What needs to be changed?

1. Write it down instead of speak it out
    + Not everyone is comfortable with it on Day 1.
    + It takes us about a week to get used to writing doc instead of Zoom calling or Slack chatting.
2. Sending links of doc, along with some copy and paste
    + Other teams may still prefer more traditional email communication.
    + No hard feeling. Just send them the link along with some content of the doc copy and pasted into the email. 
3. Higher throughput, but more latency
    + The async nature of merge request flow means we won't get response as fast as tapping one's shoulder in office.
    + But the upside is higher quality communication and easier to discuss mutliple things without getting the context mixed up. Communication threads in merge requests are precise to a single line.



### Conclusion
It is an efficient, impartial, system.
It requires some technical capability, but really not much.
You may encounter resistence, but even if you can only successfully promote it within your team. The benefit is still worths it.

By the way, the [GitLab Handbook](https://about.gitlab.com/handbook/){:target="_blank"} is really a valuable reference to how to run different departments of a software company there. I am so happy that they are able to condense their valuable experience into a single document and open source it.



### References
1. [We’ve been trained to make paper](https://ben.balter.com/2012/10/19/we-ve-been-trained-to-make-paper/){:target="_blank"}  
2. [GitLab Handbook](https://about.gitlab.com/handbook/){:target="_blank"}  
3. [Markdown Emoji](https://www.webfx.com/tools/emoji-cheat-sheet/){:target="_blank"}  
4. [Source code of this blog](https://github.com/billykong/billykong.github.io){:target="_blank"}
5. [GitLab Merge Request Flow](https://docs.gitlab.com/ee/user/project/merge_requests/){:target="_blank"}
6. [GitHub Pull Request Flow](https://guides.github.com/introduction/flow/){:target="_blank"}

<br>
<br>

[gitlab]: https://gitlab.com
[github]: https://github.com
[emoji]: https://www.webfx.com/tools/emoji-cheat-sheet/

