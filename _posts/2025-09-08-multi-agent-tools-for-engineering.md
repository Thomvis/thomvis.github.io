---
layout: post
title: "Building multi-agent tools for engineering"
---

I wrote a post about some of the work I've been doing at NewStore:

> In this post we’ll discuss two multi-agent tools:
> - A Slack-bot that helps us triage reported issues with our shopping apps
> - A research agent that helps us respond to incidents [...]
> 
> Our first multi-agent tool builds upon the testing agent we introduced in a [previous post](https://newstoretech.substack.com/p/testing-our-shopping-apps-with-ai). [...] 
> To unlock the agent for use in a multi-agent system, we updated the CI job to check for the presence of a “custom instructions” environment variable.
> We thought it would be really useful if we could trigger these custom test runs from within Slack. [...] To achieve this we introduced a second agent as the conversational front-end of the tool. We’ve named it Shoppy. [...]
>  
> The incident research agent surprised us with its performance. With every tool we gave it, it seemed to get better at its task. There are still a lot of tools to add to give it insight into other major parts of our technology stack. It’ll be interesting to see if the agent’s performance will continue to scale with an even larger number of tools at its disposal.

Read the full post over at the [NewStore Tech Blog](https://newstoretech.substack.com/p/building-multi-agent-tools-for-engineering).