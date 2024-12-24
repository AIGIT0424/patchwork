Based on the code, here's the updated README documentation:

<div align="center">
  <picture>
    <img alt="Patchwork logo" src="https://repository-images.githubusercontent.com/782544882/a9743f35-5e1c-43ed-a0e0-536322056d38" width="36%">
  </picture>
 <br>
  <img alt="Patchwork GIF" src="https://raw.githubusercontent.com/patched-codes/patchwork/main/patchwork-banner.gif">
</div>
<br>

<div align="center">

[![Build](https://github.com/patched-codes/patchwork/actions/workflows/release.yml/badge.svg)](https://github.com/patched-codes/patchwork/actions/workflows/release.yml)
[![Discord](https://dcbadge.limes.pink/api/server/XDxA3mJyhE?style=flat&theme=clean-inverted)](https://discord.gg/XDxA3mJyhE)
[![Downloads](https://img.shields.io/pypi/v/patchwork-cli)](https://pypi.org/project/patchwork-cli/)
[![Downloads](https://static.pepy.tech/badge/patchwork-cli)](https://pepy.tech/projects/patchwork-cli)

[Demo](https://youtu.be/MLyn6B3bFMU) |
[Docs](https://docs.patched.codes/)

</div>

Patchwork automates development tasks like PR reviews, bug fixing, security patching, and more using a self-hosted CLI agent and your preferred LLMs. Try the hosted version [here](https://app.patched.codes/signin).

## Key Components

- **Steps**: Reusable atomic actions like create PR, commit changes or call an LLM. Steps are implemented as Python classes that inherit from a base Step class and define inputs/outputs through TypedDict.
- **Prompt Templates**: Customizable LLM prompts optimized for specific tasks like code generation, issue analysis or vulnerability remediation. Templates use placeholders in {{}} format that get replaced with data.
- **Patchflows**: LLM-assisted automation workflows built by combining steps and prompts. Patchflows are Python classes that inherit from Patchflow and define required steps.

Patchflows can be run locally in your CLI and IDE, or as part of your CI/CD pipeline. The framework comes with several built-in patchflows and allows creating custom ones.

## Demo

[![Patchwork CLI Quickstart](https://img.youtube.com/vi/3gRpqQoIino/0.jpg)](https://youtu.be/MLyn6B3bFMU)

## Installation

### Using Pip

```bash
pip install 'patchwork-cli[all]' --upgrade
```

Optional dependency groups:

- `security`: Installs `semgrep` and `depscan` for vulnerability scanning
- `rag`: Installs `chromadb` for embedding and retrieval
- `notifications`: For notification features like Slack messages  
- `all`: Installs everything
- Core install runs GenerateDocstring, PRReview and GenerateREADME flows

### Using Poetry

See [INSTALL.md](INSTALL.md) for building from source with poetry.

## CLI Usage

```bash
patchwork <PatchFlow> <?Arguments>
```

Arguments override patchflow defaults in `key=value` format. Boolean flags take no value.

Example (AutoFix with OpenAI):
```bash
patchwork AutoFix openai_api_key=<KEY> github_api_key=<TOKEN>
```

Use Patched managed API:
```bash 
patchwork AutoFix patched_api_key=<KEY> github_api_key=<TOKEN>
```

Use Google models:
```bash
patchwork AutoFix google_api_key=<KEY> model=gemini-pro-1.5
```

Custom configs:
```bash
patchwork AutoFix --config /path/to/configs
```

## Open Source Models

Supports any OpenAI-compatible endpoint (Groq, Together AI, HuggingFace etc).

Using Groq:
```bash
patchwork AutoFix client_base_url=https://api.groq.com/openai/v1 openai_api_key=<KEY> model=llama-3.1-405b-reasoning
```

HuggingFace via config.yml:
```yaml
openai_api_key: <KEY>
client_base_url: https://api-inference.huggingface.co/models/meta-llama/Meta-Llama-3.1-405B-Instruct-FP8/v1 
model: Meta-Llama-3.1-405B-Instruct-FP8
```

Run locally via llama.cpp:
```bash
python -m llama_cpp.server --hf_model_repo_id bullerwins/Meta-Llama-3.1-8B-Instruct-GGUF --model 'Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf' --chat_format chatml

patchwork AutoFix client_base_url=http://localhost:8080/v1 openai_api_key=no_key_local_model
```

## Built-in Patchflows

- **GenerateDocstring**: Auto-generate docstrings for methods
- **AutoFix**: Fix code vulnerabilities found by scanners
- **PRReview**: Review PRs by analyzing diffs
- **GenerateREADME**: Create README documentation
- **DependencyUpgrade**: Update dependencies to fixed versions
- **ResolveIssue**: Find and fix reported issues
- **GenerateCodeUsageExample**: Generate code examples
- **GenerateDiagram**: Create code diagrams
- **GenerateUnitTests**: Generate unit tests
- **SonarFix**: Fix SonarQube findings

## Prompt Templates 

JSON templates define LLM interactions:

```json
{
  "id": "diffreview_summary",
  "prompts": [
    {
      "role": "user", 
      "content": "Summarize the code changes: {{diffreviews}}"  
    }
  ]
}
```

Each patchflow has default templates but you can override with:
```bash
patchwork AutoFix prompt_template_file=/path/to/template.json
```

## Contributing

See [patchwork/patchflows/README.md](patchwork/patchflows/README.md) for creating patchflows and [patchwork/steps/README.md](patchwork/steps/README.md) for creating steps.

Chat assistant available at [HuggingChat](https://hf.co/chat/assistant/66322701fd4787e0c1f7696b).

## License

AGPL-3.0 for core framework, Apache-2.0 for custom patchflows and steps.
