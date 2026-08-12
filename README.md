# Building with the Claude API

This repository contains my notes and exercises from the Anthropic course **"Building with the Claude API"**.

## Setup

1. Clone this repository.
2. Create a virtual environment and install the dependencies (e.g. `anthropic`, `python-dotenv`).
3. Create a `.env` file in the project root with your configuration (see below).
4. Run the notebooks/scripts.

## Environment Variables

Create a `.env` file in the root of the project with the following fields:

```dotenv
ANTHROPIC_BASE_URL=
ANTHROPIC_AUTH_TOKEN=
ANTHROPIC_MODEL="anthropic.claude-4-6-sonnet"
ANTHROPIC_DEFAULT_HAIKU_MODEL="anthropic.claude-4-5-haiku"
ANTHROPIC_DEFAULT_SONNET_MODEL="anthropic.claude-4-6-sonnet"
ANTHROPIC_DEFAULT_OPUS_MODEL="anthropic.claude-4-6-opus"
CLAUDE_CODE_SKIP_AUTH_LOGIN="true"
CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS="1"
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS="1"
```

- `ANTHROPIC_BASE_URL`: base URL of the Anthropic API endpoint (or proxy) you are using.
- `ANTHROPIC_AUTH_TOKEN`: authentication token used to access the API.
- `ANTHROPIC_MODEL`: default model used in requests.
- `ANTHROPIC_DEFAULT_HAIKU_MODEL` / `ANTHROPIC_DEFAULT_SONNET_MODEL` / `ANTHROPIC_DEFAULT_OPUS_MODEL`: default models for each Claude tier.
- `CLAUDE_CODE_*`: configuration flags used by Claude Code.

> **Note:** Never commit your `.env` file. Make sure it is listed in `.gitignore`.

## Usage

Load the environment variables in your notebook/script before creating the client:

```python
import os
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()

client = Anthropic(
    api_key=os.environ["ANTHROPIC_AUTH_TOKEN"],
    base_url=os.environ["ANTHROPIC_BASE_URL"],
)

model = os.environ["ANTHROPIC_MODEL"]
```
