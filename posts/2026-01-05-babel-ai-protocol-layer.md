Title: Babel: One Interface for Every AI Provider
Date: 2026-01-05
Tags: clojure, ai, openai, anthropic, gemini
Preview: true

The AI model landscape is fragmented. OpenAI, Anthropic, Google, Azure - each with their own API conventions, message formats, tool calling schemas, and streaming protocols. If you want resilience through multi-provider routing, you end up writing adapter code for each one. Then you maintain it as APIs evolve.

[Babel](https://github.com/parenstech/babel) is the protocol layer that unifies them. Define your tools once, execute them anywhere. Canonicalize messages into a single format, transform them to provider-specific wire formats on the way out. Stream responses through a consistent interface regardless of whether the underlying protocol uses SSE or WebSockets.

The library handles the gnarly details: prompt caching strategies that differ by provider, vision and PDF input normalization, extended thinking for Claude, image generation, circuit breakers with automatic failover. Same application code, different provider by changing a keyword. When one provider is down or rate-limited, requests route to the next.

This is not about abstracting away the differences - each provider has unique capabilities worth using. It is about having a stable foundation so you can focus on what your application does rather than how to talk to the models.
