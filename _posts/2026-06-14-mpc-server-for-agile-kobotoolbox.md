---
title: "MCP Server"
date: 2026-06-14
categories: [ai, mpc servers, kobotoolbox, agile, python, claude, claude desktop, fastmcp]
---

So i tried to visit one of my throwaway dashboards and i forgot the password i had set. I could easily check where i had it saved but i could not be bothered at that time. But that got me thinking, thats only useful when i am able to remember the password, and that is a bottle neck to getting information quickly. I have two ideas, i could either host it on my homelab where it wouldnt need securing cause it was for personal use anyway, or i could build an MCP server? accessible on my desktop anytime. I could ask Claude in natual language and it would give me answers back.

The MCP was more challening cause i had never done it so it would be interesting cause just basic python functions wrapped in decorators when written in python. So i got to work and went on the fastMCP website to understand.

# What is an MCP?
I will define it for fellow developers, its just a fancy name for formatted data, basic functions, usually with an API call within them that returns structured data. Why an MCP though? Well, its a standardised protocol that AI agents use to communicate with different applications. Mine would only be reading data but they can also perform tasks based on data given to the agent. It checks which 'tool' it needs and calls that tool, that tool then does what it does. Internally, the tool could even just return a "hello world" string, but that wouldnt be very useful. haha
It usually calls an API, formats the result to match what the tool name expects and returns


I had an idea, why not make an 