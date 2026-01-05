Title: Atlas: Infrastructure Visibility for Azure AI
Date: 2026-01-05
Tags: clojure, azure, infrastructure, ai
Preview: true

Managing AI infrastructure across multiple Azure subscriptions is like navigating without a map. Which models are deployed where? How much quota remains? What is available in each region? The Azure portal answers these questions, but slowly, one subscription at a time.

[Atlas](https://github.com/parenstech/atlas) is a CLI-first tool that provides unified visibility into Azure AI Foundry deployments. One command lists all models across six subscriptions. Another shows quota usage per region. A third discovers which models are available where you need them. No clicking through portals, no context switching, no forgetting which subscription holds what.

The value is simple: when you are routing AI requests across multiple deployments for resilience and cost, you need to see everything at once. Atlas gives you that view from the command line, where infrastructure decisions belong.
