# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
pipenv install

# Run the Flask app (http://localhost:5000)
pipenv run python app.py

# Lint
pipenv run pylint <file>

# Format code
pipenv run black .
pipenv run isort .

# Run tests
pipenv run pytest .
```

## Environment Setup

Copy `.env.example` to `.env` and populate:
- `OPENAI_API_KEY` — required
- `SCRAPIN_API_KEY` — required for LinkedIn data
- `TAVILY_API_KEY` — required for agent search
- `TWITTER_API_KEY/SECRET/TOKENS` — optional; app falls back to mock tweets from a GitHub Gist
- `LANGCHAIN_TRACING_V2` / `LANGCHAIN_API_KEY` / `LANGCHAIN_PROJECT` — optional LangSmith tracing

## Architecture

This is an AI-powered ice breaker generator: given a person's name, it searches for their LinkedIn and Twitter profiles and generates a summary, topics of interest, and conversation starters.

**Pipeline flow** (`ice_breaker.py:ice_break_with`):

1. **LinkedIn Lookup Agent** (`agents/linkedin_lookup_agent.py`) — ReAct agent (GPT-4o-mini + Tavily search) that finds the person's LinkedIn URL
2. **LinkedIn Scraper** (`third_parties/linkedin.py`) — Calls Scrapin.io API to fetch profile JSON
3. **Twitter Lookup Agent** (`agents/twitter_lookup_agent.py`) — Same pattern, finds Twitter handle
4. **Twitter Fetcher** (`third_parties/twitter.py`) — Tweepy client; falls back to mock tweet data from a GitHub Gist
5. **Three LangChain chains** (`chains/custom_chains.py`) — Run in parallel on the combined profile data:
   - Summary chain (GPT-3.5-turbo, temp=0) → `Summary` Pydantic model
   - Topics-of-interest chain (GPT-3.5-turbo, temp=0) → `TopicOfInterest` Pydantic model
   - Ice-breaker chain (GPT-3.5-turbo, temp=1) → `IceBreaker` Pydantic model
6. **Flask API** (`app.py`) — `POST /process` returns JSON with `summary_and_facts`, `interests`, `ice_breakers`, and `picture_url`

**Output models** are defined in `output_parsers.py` using Pydantic. Chains use LangChain Expression Language (LCEL) `RunnableSequence` and `PydanticOutputParser`.

**Frontend** (`templates/index.html`) — plain HTML + vanilla JS; submits the name form, shows a spinner, and renders the JSON response.
