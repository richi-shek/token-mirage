# Token Mirage

**Billions of tokens. Zero inference.**

Token Mirage is an open-source mock LLM API server that simulates extremely high token throughput with minimal computational resources and bandwidth.

It supports OpenAI-compatible and Anthropic-compatible APIs for testing, development, benchmarking client behavior, and entertainment.

> [!WARNING]
> **SIMULATED TOKEN USAGE — NOT REAL COMPUTATION**
>
> All reported token usage is synthetic. Token Mirage does not perform the corresponding LLM inference or generate the claimed number of real model tokens.
>
> **Do not use Token Mirage to fraudulently claim computational work, obtain virtual currency, earn API credits, manipulate reward systems, or misrepresent real AI usage.**
>
> Token Mirage is not a proof-of-computation or proof-of-inference system.

## Detection and Transparency

Token Mirage intentionally exposes recognizable identifiers so that third-party applications can distinguish simulated usage from genuine model inference.

Official implementations provide:

- `X-Token-Mirage: simulated`
- `X-Token-Mirage-Usage: synthetic`
- `X-Token-Mirage-Inference: none`
- `metadata.token_mirage: "simulated"` in OpenAI Responses
- SSE simulation comments
- `/.well-known/token-mirage.json`

These indicators are designed to help identify official or unmodified installations. They are not cryptographic proof and may be removed by modified implementations or network intermediaries.

**Reward and billing platforms must not trust arbitrary API usage reports as proof of actual inference.** They should independently verify usage through trusted providers and authenticated billing records.

## License

Licensed under the **GNU Affero General Public License v3.0 only (AGPL-3.0-only)**.

The responsible-use statement above describes the project's intended use and does not introduce additional license restrictions.
