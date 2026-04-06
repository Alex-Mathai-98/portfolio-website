---
categories: [Technical]
math: true
---

# Using Gemini Models Inside Claude Code via LiteLLM

How to route Claude Code's CLI through a LiteLLM proxy so it talks to Vertex AI (Gemini) on the backend --- with both Google AI Studio and Vertex AI options.

---

## 1. Why Would You Do This?

Claude Code is a solid agentic coding CLI. But sometimes you want to run a different model behind it:

- **Stay within your GCP billing** instead of paying Anthropic separately
- **Compare model outputs** on the same coding tasks
- **Use a different model's context window** for specific workloads

The trick is that Claude Code speaks the Anthropic Messages API. Gemini speaks Google's API. You need a translator in the middle. That's where **LiteLLM** comes in.

---

## 2. The Architecture

```
You (terminal)
    ↓
Claude Code CLI
    ↓  (Anthropic Messages API format)
LiteLLM Proxy (localhost:4000)
    ↓  (translates to Google's API format)
Vertex AI / Google AI Studio
    ↓
Gemini model (3.1 Pro, 2.5 Flash, etc.)
```

LiteLLM is an open-source proxy that accepts requests in one API format and forwards them to 100+ LLM providers. Here, we configure it to accept Anthropic-format requests from Claude Code and translate them into Vertex AI calls to Gemini.

---

## 3. Prerequisites

- **Claude Code** installed ([docs](https://docs.anthropic.com/en/docs/claude-code/overview))
- **Python 3.10+** with pip
- One of:
  - **Google AI Studio API key** (Option A --- simpler, free tier available)
  - **GCP project with Vertex AI enabled** (Option B --- uses your existing billing)

---

## 4. Install LiteLLM

```bash
pip install 'litellm[proxy]'
```

This installs both the LiteLLM library and its proxy server.

---

## 5. Choose Your Provider

You have two options for how Gemini receives requests. Pick one.

### Option A: Google AI Studio (Simpler)

Best for: quick setup, personal use, free tier.

1. Get an API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. Export it:

```bash
export GEMINI_API_KEY="your-key-here"
```

### Option B: Vertex AI (GCP)

Best for: existing GCP projects, enterprise billing, higher rate limits.

1. Authenticate with GCP:

```bash
gcloud auth application-default login
```

2. Set your project:

```bash
export GOOGLE_CLOUD_PROJECT="your-gcp-project-id"
```

No API key needed --- Vertex AI uses Application Default Credentials (ADC).

---

## 6. Create the LiteLLM Config

Create a file at `~/.litellm/config.yml`:

### Option A config (Google AI Studio):

```yaml
litellm_settings:
  drop_params: true
  set_verbose: true

model_list:
  # Primary model
  - model_name: gemini-3.1-pro-preview
    litellm_params:
      model: gemini/gemini-3.1-pro-preview
      api_key: os.environ/GEMINI_API_KEY

  # Faster/cheaper option
  - model_name: gemini-2.5-flash
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY

  # Budget option
  - model_name: gemini-3.1-flash-lite
    litellm_params:
      model: gemini/gemini-3.1-flash-lite-preview
      api_key: os.environ/GEMINI_API_KEY
```

### Option B config (Vertex AI):

```yaml
litellm_settings:
  drop_params: true
  set_verbose: true

model_list:
  - model_name: gemini-3.1-pro-preview
    litellm_params:
      model: vertex_ai/gemini-3.1-pro-preview
      vertex_project: "your-gcp-project-id"
      vertex_location: "global"

  - model_name: gemini-2.5-flash
    litellm_params:
      model: vertex_ai/gemini-2.5-flash
      vertex_project: "your-gcp-project-id"
      vertex_location: "global"

  - model_name: gemini-3-flash-preview
    litellm_params:
      model: vertex_ai/gemini-3-flash-preview
      vertex_project: "your-gcp-project-id"
      vertex_location: "global"

  - model_name: gemini-3.1-flash-lite
    litellm_params:
      model: vertex_ai/gemini-3.1-flash-lite-preview
      vertex_project: "your-gcp-project-id"
      vertex_location: "global"
```

**Key settings explained:**

| Setting | Purpose |
|---|---|
| `drop_params: true` | Silently drops parameters that Gemini doesn't support (e.g., Anthropic-specific fields) instead of erroring |
| `set_verbose: true` | Enables detailed logging --- useful for debugging, disable once stable |
| `model: vertex_ai/...` | The `vertex_ai/` prefix tells LiteLLM to use the Vertex AI provider |
| `model: gemini/...` | The `gemini/` prefix tells LiteLLM to use the Google AI Studio provider |
| `vertex_location: "global"` | Routes to Google's global endpoint; you can also use `us-central1`, `europe-west4`, etc. |

---

## 7. Start the Proxy

Set a master key (this authenticates Claude Code to the proxy):

```bash
export LITELLM_MASTER_KEY="sk-litellm-$(openssl rand -hex 16)"
echo "Your master key: $LITELLM_MASTER_KEY"
```

Start the proxy:

```bash
litellm --config ~/.litellm/config.yml
```

The proxy starts on `http://0.0.0.0:4000`. You should see it log the available models.

### Verify it works (in a new terminal):

```bash
curl -X POST http://0.0.0.0:4000/v1/messages \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-3.1-pro-preview",
    "max_tokens": 200,
    "messages": [{"role": "user", "content": "Say hello"}]
  }'
```

If you get a response back, the proxy is working.

---

## 8. Connect Claude Code

In a new terminal, point Claude Code at the proxy:

```bash
export ANTHROPIC_BASE_URL="http://0.0.0.0:4000"
export ANTHROPIC_AUTH_TOKEN="$LITELLM_MASTER_KEY"
export CLAUDE_CODE_ATTRIBUTION_HEADER=0
```

Now launch Claude Code with a Gemini model:

```bash
claude --model gemini-3.1-pro-preview

# Or a lighter model
claude --model gemini-2.5-flash
```

That's it. Claude Code's full toolset --- file editing, bash execution, glob/grep, multi-step planning --- now runs on Gemini under the hood.

---

## 9. Make It Permanent (Optional)

Add aliases to your `~/.zshrc` or `~/.bashrc` so you can switch between Claude and Gemini effortlessly:

```bash
# Claude Code with Gemini backend
alias claude-gemini='ANTHROPIC_BASE_URL="http://0.0.0.0:4000" \
  ANTHROPIC_AUTH_TOKEN="your-master-key" \
  CLAUDE_CODE_ATTRIBUTION_HEADER=0 \
  claude --model gemini-3.1-pro-preview'

alias claude-flash='ANTHROPIC_BASE_URL="http://0.0.0.0:4000" \
  ANTHROPIC_AUTH_TOKEN="your-master-key" \
  CLAUDE_CODE_ATTRIBUTION_HEADER=0 \
  claude --model gemini-2.5-flash'
```

Then you can just type `claude-gemini` or `claude-flash` from any terminal.

---

## 10. Model Selection Guide

Which Gemini model to use depends on your task:

| Model | Use Case | Speed | Cost |
|---|---|---|---|
| **Gemini 3.1 Pro** | Multi-file refactors, architecture questions | Slower | Higher |
| **Gemini 2.5 Flash** | General coding, quick edits | Fast | Lower |
| **Gemini 3 Flash** | General purpose | Fast | Lower |
| **Gemini 3.1 Flash Lite** | Simple queries, high-volume tasks | Fastest | Lowest |

---

## 11. Troubleshooting

**"Connection refused" errors:**
Make sure the LiteLLM proxy is running in another terminal. The proxy must stay alive for the duration of your Claude Code session.

**"Model not found" errors:**
Check that the `model_name` in your config exactly matches what you pass to `claude --model`. The names are case-sensitive.

**Authentication failures (Vertex AI):**
Run `gcloud auth application-default login` again. ADC tokens expire --- if you haven't authenticated recently, refresh them.

**Unsupported parameter warnings:**
This is normal. Claude Code sends some Anthropic-specific parameters that Gemini doesn't understand. The `drop_params: true` setting handles this gracefully, but you may still see warnings in verbose mode.

**Slow first request:**
The first request through the proxy takes longer because LiteLLM needs to establish the connection. Subsequent requests are faster.

---

## 12. What Works and What Doesn't

Claude Code's tooling layer is model-agnostic --- it handles tool calls, file operations, and shell execution regardless of which model generates the instructions. In practice:

**Works well:**
- File reading, editing, and creation
- Bash command execution
- Glob and grep searches
- Multi-step task planning
- Code generation and refactoring

**May degrade:**
- Complex tool-use chains (Gemini occasionally formats tool calls differently than Claude expects)
- Very long conversations (token counting differences between providers)
- Some Claude-specific features like extended thinking

The experience works for most coding tasks. The main trade-off is that you lose native Anthropic API compatibility in exchange for provider flexibility.

---

*The LiteLLM proxy approach works with any provider LiteLLM supports --- not just Gemini. The same pattern applies to OpenAI, Mistral, Deepseek, or any other model you want to run through Claude Code's interface.*
