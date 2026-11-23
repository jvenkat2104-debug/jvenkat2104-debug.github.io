# LiteLLM

**Level: Beginner** · Basic PowerShell and HTTP knowledge helps.

Status: Draft | Model listing and end-to-end chat confirmed in a learner run

LiteLLM is an open-source project from BerriAI that gives applications a common API for calling models across providers. It comes as a Python SDK you can import into an application and as a proxy you can run as a separate gateway service. [LiteLLM overview](https://docs.litellm.ai/docs/) · [Wikipedia background](https://en.wikipedia.org/wiki/LiteLLM)

## Where LiteLLM fits

Imagine you’re building a college canteen assistant. You start with a model on your laptop and later want to try a model hosted on AWS. With the LiteLLM proxy, the app can send the same basic chat request to an OpenAI-compatible endpoint. The proxy checks the key and uses the requested model alias to route the call to the configured backend.

| Name | What it does |
| --- | --- |
| Qwen, Claude, GPT | Examples of large language model families. A model generates the response. |
| Ollama | Runs the downloaded Qwen model on your computer and exposes it over HTTP. |
| Amazon Bedrock | A managed AWS service that gives applications API access to hosted foundation models. |
| LiteLLM Proxy | Gives clients one API endpoint and routes their requests to a configured backend, such as Ollama or Bedrock. |

If your application only uses models through Bedrock, it can call Bedrock directly. A gateway can be useful when several applications need a common interface across local and cloud backends. LiteLLM can sit in front of Bedrock, but swapping backends can still change response quality, latency, and supported features. Test the prompts and features your app depends on after a switch. [Amazon Bedrock overview](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) · [LiteLLM Bedrock integration](https://docs.litellm.ai/docs/providers/bedrock)

For a shared gateway, LiteLLM can also issue virtual keys to callers, track usage and spend, and apply configured budgets or rate limits. In this lab, you’ll start with one key and see how aliases and fallback work before adding those controls. [LiteLLM overview](https://docs.litellm.ai/docs/)

## Run a local LiteLLM gateway on Windows

```text
PowerShell → LiteLLM Proxy (Docker) → Ollama (Windows) → Qwen2.5 3B
```

You’ll first ask Qwen a question directly, then send a request through the gateway. This gives you a working checkpoint before you add Docker networking and gateway authentication. The lab uses Windows and Docker Desktop with WSL 2. You need internet access for the downloads and enough resources to run the model. The model download is about 1.9 GB. Inference uses your local machine and requires no cloud API key. [Ollama model page](https://ollama.com/library/qwen2.5:3b)

From Windows, PowerShell calls LiteLLM on `127.0.0.1:4000`. From inside Docker, LiteLLM calls Ollama on `host.docker.internal:11434`. The special hostname lets a Docker Desktop container reach a service on the host. [Docker Desktop networking](https://docs.docker.com/desktop/features/networking/) · [LiteLLM Ollama integration](https://docs.litellm.ai/docs/providers/ollama)

## 1. Verify WSL 2 and Docker

In PowerShell, run `wsl --status` and `wsl --list --verbose`. If WSL is missing, run `wsl --install` from an administrator PowerShell and restart if prompted. Open Docker Desktop with the WSL 2 engine enabled. Then run:

```powershell
docker version
docker run --rm hello-world
```

Continue when `docker version` shows a server and the test container prints `Hello from Docker!`. If the server is unavailable, check that Docker Desktop is running before changing the LiteLLM configuration. [Microsoft WSL installation](https://learn.microsoft.com/en-us/windows/wsl/install) · [Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)

## 2. Run the model directly

Install and start [Ollama for Windows](https://docs.ollama.com/windows). In PowerShell, run:

```powershell
ollama --version
ollama pull qwen2.5:3b
ollama run qwen2.5:3b
```

Ask the model to explain the difference between a model, an inference server and an AI gateway in three sentences. Once it responds, type `/bye` to leave the interactive prompt. Keep Ollama running and check its HTTP API:

```powershell
(Invoke-RestMethod 'http://127.0.0.1:11434/api/tags').models |
    Select-Object -ExpandProperty name
```

Look for `qwen2.5:3b`. This checks that Ollama responds on the Windows host and knows about the model.

## 3. Configure and start LiteLLM

Before pulling a Docker image or installing the Python package, check the project’s security advisories. For a repeatable lab, pin the image you’ve reviewed by digest so later runs use the same artifact. The `latest` tag can move. [LiteLLM security advisories](https://github.com/BerriAI/litellm/security)

Create a separate lab folder and save the following as `config.yaml`. If you use Notepad, check that the filename ends in `.yaml`.

```yaml
model_list:
  - model_name: local-assistant
    litellm_params:
      model: ollama_chat/qwen2.5:3b
      api_base: http://host.docker.internal:11434

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

`local-assistant` is the name the client sends. The gateway uses the configuration to select the actual provider and model. [LiteLLM Proxy configuration](https://docs.litellm.ai/docs/proxy/configs)

Open PowerShell in the lab folder. Generate a key without printing it, then start Docker in detached mode so the same shell stays available for the API requests.

```powershell
$config = (Resolve-Path '.\config.yaml').Path
$bytes = New-Object byte[] 32
$rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
try {
    $rng.GetBytes($bytes)
} finally {
    $rng.Dispose()
}
$env:LITELLM_MASTER_KEY = 'sk-' + [BitConverter]::ToString($bytes).Replace('-', '')

docker run --detach --rm --name litellm-lab `
  --publish 127.0.0.1:4000:4000 `
  --mount "type=bind,source=$config,target=/app/config.yaml,readonly" `
  --env LITELLM_MASTER_KEY `
  docker.litellm.ai/berriai/litellm:latest `
  --config /app/config.yaml

docker logs --tail 30 litellm-lab
```

The port binding limits the gateway to this machine. The image tag follows the lab example. Record the image digest if you need to reproduce the same run later. [LiteLLM deployment documentation](https://docs.litellm.ai/docs/proxy/deploy)

## 4. List the model and send a request

Use the **same PowerShell window**. Once the gateway is ready, verify that it lists the alias:

```powershell
$headers = @{ Authorization = "Bearer $env:LITELLM_MASTER_KEY" }
$base = 'http://127.0.0.1:4000'

(Invoke-RestMethod "$base/v1/models" -Headers $headers).data |
    Select-Object -ExpandProperty id
```

Seeing `local-assistant` checks that the gateway accepts the key and exposes the alias. Send a chat request to check the full path through Ollama:

```powershell
$body = @{
    model = 'local-assistant'
    messages = @(
        @{ role = 'user'; content = 'Explain why a college canteen tracks cancelled orders. Use two sentences.' }
    )
} | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod "$base/v1/chat/completions" `
    -Method Post -Headers $headers -ContentType 'application/json' `
    -Body $body -TimeoutSec 120

$response.choices[0].message.content
```

A successful request prints model-generated text. Start with a synthetic prompt and expect the wording to vary. The first request can take longer while Ollama loads the model.

### What a successful chat request demonstrates

- **Model alias.** PowerShell sends `local-assistant`. The gateway resolves it to the configured Qwen model, so a backend change can be made in gateway configuration.
- **Gateway access and routing.** LiteLLM checks the bearer key and forwards the request from `/v1/chat/completions` to Ollama at `host.docker.internal:11434`.
- **Inference.** Ollama runs Qwen and generates the answer. The response travels back through LiteLLM to PowerShell.

## Troubleshoot one layer at a time

| Symptom | First check |
| --- | --- |
| A new PowerShell window has no key | Environment variables set this way belong to the original shell and its child processes. Use the original window. If it closed, stop the lab container and restart it with a newly generated key. |
| Gateway returns HTTP 401 | Check whether the variable is set with `[bool]$env:LITELLM_MASTER_KEY`. If you generated a new key after starting the container, restart the lab with that key. |
| Gateway exits with `yaml.parser.ParserError` after removing a model | Align the remaining `- model_name:` two spaces under `model_list:` and use four spaces for its sibling `litellm_params:`. Remove fallback rules that reference a deleted alias. |
| Gateway returns HTTP 400 with `Invalid model name` | Check `/v1/models` with the same key. If you removed `local-assistant` and kept only Llama, update the request to use `fast-assistant` and rebuild the JSON body. |
| Port 4000 refuses connections | Run `docker ps` and check the gateway logs. If the container exited and Docker removed it, rerun the original Docker command without `--detach` to see the startup error in the terminal. |
| Ollama responds on Windows, but chat through LiteLLM fails | Check the gateway logs and confirm the configured host is `host.docker.internal`. From Docker Desktop, check whether a container can reach Ollama's `/api/tags` endpoint before changing any listener or firewall settings. |
| The response is slow | The first request may need to load the model. Try the same short prompt twice and record how long each request takes. |

Keep the master key out of screenshots, shell transcripts and shared notes. To finish the lab, run `docker stop litellm-lab` and `Remove-Item Env:LITELLM_MASTER_KEY` in the shell where you set it.

In the learner run, a request to the removed `local-assistant` alias returned HTTP 400. Printing `$response` on a separate line then showed text from an earlier request. Clear `$response` before retrying and keep the output statement inside the same `try` block as the request, with `-ErrorAction Stop`.

## Record what happened

The learner confirmed the WSL setup worked, Docker ran, Ollama returned a model response and LiteLLM started. After troubleshooting a missing key in a second PowerShell window, the authenticated request to `/v1/models` listed `local-assistant`. The walkthrough uses detached Docker so both the gateway and client can use the same shell session.

The learner then sent “Explain the role of an AI gateway in enterprise architecture in 3 sentences” to `/v1/chat/completions`. Qwen returned a three-sentence answer describing how a gateway connects applications to AI services and standardizes requests. This confirmed the full request and response path. The answer also claimed a gateway ensures privacy and security. Treat that as a model-generated claim. This lab exercised bearer-key authentication on localhost. Privacy and security in a deployed gateway depend on its configuration and operating practices.

For each checkpoint, record the tool versions, the command, the expected result and what you observed. A redacted error and the step that fixed it are enough to make a useful troubleshooting entry.

## Lab 2 — Add a second local model

Ollama already has an API, so why add LiteLLM? A client can call Ollama directly when that meets its needs. In Lab 1, the gateway gave the client a stable model alias. For example, the client could send:

```json
{
  "model": "local-assistant",
  "messages": [
    {"role": "user", "content": "Explain cloud computing."}
  ]
}
```

Today the alias maps to Qwen. You can later update the gateway configuration to point it at another supported backend while keeping the same request format for a basic chat call. This is **loose coupling**. The client depends on the gateway’s interface, and the gateway configuration determines where inference runs. Retest the client’s prompts and required features after changing the backend.

### Download and verify Llama 3.2

With Ollama running, use PowerShell:

```powershell
ollama pull llama3.2:1b
ollama list
```

Look for both `qwen2.5:3b` and `llama3.2:1b`. The learner confirmed both models are installed in Ollama. [Ollama model page](https://ollama.com/library/llama3.2:1b)

### Register both models in LiteLLM

Open PowerShell in the same lab folder you used for Lab 1 and run `docker stop litellm-lab` if the container is still running. Replace the contents of `config.yaml` with:

```yaml
model_list:
  - model_name: local-assistant
    litellm_params:
      model: ollama_chat/qwen2.5:3b
      api_base: http://host.docker.internal:11434

  - model_name: fast-assistant
    litellm_params:
      model: ollama_chat/llama3.2:1b
      api_base: http://host.docker.internal:11434

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Rerun the detached Docker command from [Configure and start LiteLLM](#3-configure-and-start-litellm). Reuse the key if it is still set in your PowerShell session. If you opened a new session, generate a new key using that section before starting Docker. Wait for the gateway to start, then run:

```powershell
$base = 'http://127.0.0.1:4000'
$headers = @{ Authorization = "Bearer $env:LITELLM_MASTER_KEY" }
(Invoke-RestMethod "$base/v1/models" -Headers $headers).data |
    Select-Object -ExpandProperty id
```

Expect `local-assistant` and `fast-assistant`, in either order. This verifies the gateway lists both aliases. To check that each alias can generate a response, run the comparison below.

### Compare both models with the same prompt

```powershell
$prompt = 'Explain why an enterprise needs an AI gateway. Give three reasons.'

foreach ($model in @('fast-assistant', 'local-assistant')) {
    $body = @{
        model = $model
        messages = @(@{ role = 'user'; content = $prompt })
        max_tokens = 150
    } | ConvertTo-Json -Depth 5

    $timer = [System.Diagnostics.Stopwatch]::StartNew()
    $response = Invoke-RestMethod "$base/v1/chat/completions" `
        -Method Post -Headers $headers -ContentType 'application/json' `
        -Body $body -TimeoutSec 120 -ErrorAction Stop
    $timer.Stop()

    [pscustomobject]@{
        Model = $model
        LatencySeconds = [math]::Round($timer.Elapsed.TotalSeconds, 2)
        Response = $response.choices[0].message.content
    } | Format-List
}
```

The timer measures elapsed time from the PowerShell client. It includes gateway and network overhead, any model loading and inference. The same token cap limits both outputs, but the models can still produce answers of different lengths. Run a second request immediately after the first for each model to see how reuse affects latency. Record the run order because switching models can change what stays loaded.

| Check | What to record |
| --- | --- |
| Latency | First and repeat request times for each alias |
| Clarity | Which answer is easier to read, and why |
| Completeness | Whether the answer gives three distinct reasons |
| Accuracy | Whether the answer makes claims the reader can support |

### Results from the learner run

Both aliases returned text through `/v1/chat/completions`. The learner ran Llama first and Qwen second, with the same prompt and a `max_tokens` limit of 150.

| Alias and model | Client-observed latency | Response quality |
| --- | ---: | --- |
| `fast-assistant` · Llama 3.2 1B | 13.48 s | Spent much of the answer on data integration and cut off during its second reason. It also overstated what a gateway guarantees about data protection. |
| `local-assistant` · Qwen2.5 3B | 20.62 s | Introduced the gateway as a bridge to AI services and described modularity, then cut off during its second reason. |

Llama finished 7.14 seconds sooner, about 35% less elapsed time relative to Qwen in this run. Neither answer delivered all three requested reasons. The 150-token cap limits generated tokens. A token can be a word, part of a word or punctuation. A later transcript showed that the last response in the loop, Qwen’s, reported `length` and 150 output tokens. Llama’s finish reason was unreported for this run.

### Check why the answers stopped

Add these lines inside the benchmark loop, just after `$timer.Stop()`, so the output captures each model’s response metadata:

```powershell
Write-Host "FINISH REASON: $($response.choices[0].finish_reason)"
Write-Host "INPUT TOKENS: $($response.usage.prompt_tokens)"
Write-Host "OUTPUT TOKENS: $($response.usage.completion_tokens)"
```

| Finish reason | How to interpret it |
| --- | --- |
| `stop` | Generation ended normally or reached a stop sequence. Check whether the answer actually delivered all three reasons. |
| `length` | Generation reached a length limit. Output usage near 150 supports the hypothesis that this request’s cap caused the cutoff. |
| `content_filter` | A content filter stopped generation, where the backend reports this reason. |

Rerun with `max_tokens = 150` to collect the missing baseline. Then run again with `max_tokens = 400`, keeping the prompt and model order the same. For each alias and cap, record latency, finish reason, input and output tokens when reported, and how many complete reasons appeared. Record blank metadata fields as unreported.

### Results at 400 tokens

| Run | Model alias | Client latency | Finish reason | Output tokens | Complete reasons |
| --- | --- | ---: | --- | ---: | ---: |
| Learner transcript | `fast-assistant` | 29.83 s | `stop` | 344 | 3 |
| Learner transcript | `local-assistant` | 64.03 s | `length` | 400 | — |
| Separate local run | `fast-assistant` | 50.93 s | `stop` | 327 | 3 |
| Separate local run | `local-assistant` | 68.10 s | `stop` | 351 | 3 |

Both aliases completed the requested three reasons in the separate run. Qwen reached the 400-token cap in the learner run. The two Llama timings also show how much elapsed time can vary between runs. Evaluate the claims in each response separately from its completeness, especially any guarantees about security or regulatory compliance.

A higher cap may let the models finish and also increase total latency if they generate more text. Compare repeated requests under similar model-loading conditions before drawing a speed conclusion. Once you’ve compared the two caps, try a separate prompt experiment that asks for exactly three bullets of at most 15 words each.

### Architecture exercise

Your enterprise has 50 internal applications. Some need quick, low-cost replies. Others need stronger reasoning. Would you let each application integrate with its preferred model directly, or put approved models behind a shared gateway?

A gateway can reduce duplicate provider integrations and give the platform team a common place for authentication, routing and usage policies. It also adds a shared dependency, so decide who owns its availability and when a direct integration is justified. Pick one application in each group and explain how you would choose its model and measure whether it meets the team’s needs.

## Lab 3A — Compare short, structured answers

Latency is how long the client waits. Output tokens divided by elapsed time gives an approximate end-to-end generation rate. It includes gateway overhead and any model loading. Serving throughput under concurrent requests would require a separate load test. Quality asks whether the answer is accurate and meets the task.

In the PowerShell session with `$headers` and `$base` set, use the same short prompt, 150-token cap and temperature for both models:

```powershell
$prompt = @'
Explain three benefits of an enterprise AI gateway.
Use exactly three numbered points.
Keep the entire answer under 60 words.
'@

foreach ($model in @('fast-assistant', 'local-assistant')) {
    $body = @{
        model = $model
        messages = @(@{ role = 'user'; content = $prompt })
        max_tokens = 150
        temperature = 0
    } | ConvertTo-Json -Depth 5

    $timer = [Diagnostics.Stopwatch]::StartNew()
    $response = Invoke-RestMethod "$base/v1/chat/completions" `
        -Method Post -Headers $headers -ContentType 'application/json' `
        -Body $body -TimeoutSec 120 -ErrorAction Stop
    $timer.Stop()

    $seconds = $timer.Elapsed.TotalSeconds
    $tokens = $response.usage.completion_tokens
    $rate = if ($tokens -gt 0 -and $seconds -gt 0) {
        [math]::Round($tokens / $seconds, 2)
    }

    [pscustomobject]@{
        Model = $model
        LatencySeconds = [math]::Round($seconds, 2)
        OutputTokens = $tokens
        FinishReason = $response.choices[0].finish_reason
        OutputTokensPerSecond = $rate
        Response = $response.choices[0].message.content
    } | Format-List
}
```

Check for exactly three numbered points, fewer than 60 words, and accurate claims. Repeat under comparable model-loading conditions before choosing a model. Compared with Lab 2, this experiment changes both the prompt and temperature, so any difference reflects the new settings together.

| Alias | Client latency | Output tokens | Finish reason | Approx. output tokens/s | Numbered points | Words |
| --- | ---: | ---: | --- | ---: | ---: | ---: |
| `fast-assistant` | 7.81 s | 76 | `stop` | 9.73 | 3 | 55 |
| `local-assistant` | 8.00 s | 53 | `stop` | 6.63 | 3 | 36 |

Both answers met the word limit in this run, counting the introductory text and omitting list numbers. Llama added a preamble and attributed data-labeling automation to the gateway, which this setup does not provide. Qwen followed the requested format more closely. Its claim about faster and more reliable processing still depends on the deployment.

### Learner repeat run

| Alias | Client latency | Output tokens | Finish reason | Approx. output tokens/s | Numbered points |
| --- | ---: | ---: | --- | ---: | ---: |
| `fast-assistant` | 7.36 s | 76 | `stop` | 10.33 | 3 |
| `local-assistant` | 5.10 s | 49 | `stop` | 9.61 | 3 |

Both answers again contained three numbered points and stayed under 60 words. Qwen finished 2.26 seconds sooner in the repeat run and generated 27 fewer tokens. Llama returned more tokens per second as measured by the client. The earlier run was nearly tied, so these observations show how answer length and elapsed time can affect the comparison. Claims about automatic workflow improvements still depend on what the application and gateway are configured to do.

## Lab 3B — Configure and verify model fallback

Keep the client request pointed at `local-assistant`. LiteLLM should send it to Qwen when Qwen is available and try Llama after an eligible primary failure. Both models share one Ollama server, so they share its availability. [LiteLLM fallback configuration](https://docs.litellm.ai/docs/proxy/reliability)

### Configure the routes

In the lab folder, stop the detached container with `docker stop litellm-lab` if it is running. Save the following as `config.yaml`. The two explicit deployment IDs make it possible to check which route served the response.

```yaml
model_list:
  - model_name: local-assistant
    litellm_params:
      model: ollama_chat/qwen2.5:3b
      api_base: http://host.docker.internal:11434
    model_info:
      id: lab-primary
  - model_name: fast-assistant
    litellm_params:
      model: ollama_chat/llama3.2:1b
      api_base: http://host.docker.internal:11434
    model_info:
      id: lab-fallback
router_settings:
  fallbacks:
    - local-assistant: [fast-assistant]
  num_retries: 0
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Start the gateway using the detached Docker command from [Configure and start LiteLLM](#3-configure-and-start-litellm). The experiment sets `num_retries: 0` so the primary request can move to fallback without a retry. Set `$base` and `$headers` as in Lab 2, using the key from the current session.

### Verify normal routing

```powershell
$body = @{
    model = 'local-assistant'
    messages = @(
        @{ role = 'user'; content = 'Explain AI gateway failover in two sentences.' }
    )
    max_tokens = 150
} | ConvertTo-Json -Depth 5

$result = Invoke-WebRequest "$base/v1/chat/completions" `
    -Method Post -Headers $headers -ContentType 'application/json' `
    -Body $body -TimeoutSec 120 -UseBasicParsing -ErrorAction Stop

$reply = $result.Content | ConvertFrom-Json
$reply.model
$result.Headers['x-litellm-model-id']
$result.Headers['x-litellm-attempted-fallbacks']
$reply.choices[0].message.content
```

Look for `lab-primary` in the deployment header. An HTTP success alone does not identify the serving model. Check the response model and gateway logs if your version omits headers. [LiteLLM response headers](https://docs.litellm.ai/docs/proxy/response_headers)

### Make the primary model unavailable

1. Stop the gateway and change only the primary `model:` value to `ollama_chat/model-does-not-exist:latest`. Keep Ollama and the Llama configuration running.
2. Restart the gateway and check that `/v1/models` still lists both `local-assistant` and `fast-assistant`. Removing the primary alias produces an invalid-model error before the router can try a fallback.
3. Send the same request. Check for deployment ID `lab-fallback` or evidence of a Llama response in the model field and logs. If the request returns an error, capture the status and inspect `docker logs litellm-lab` to see how that version classified it.
4. Restore `ollama_chat/qwen2.5:3b`, restart the gateway and verify that the same request again uses the primary.

### Verified result

Both requests used the same `local-assistant` alias. The test used temporary containers and the locally cached LiteLLM image `ef798594ca3d`.

| Primary configuration | HTTP status | Serving deployment ID | Attempted fallbacks |
| --- | ---: | --- | ---: |
| Qwen2.5 3B | 200 | `lab-primary` | 0 |
| Model missing from Ollama | 200 | `lab-fallback` | 1 |

The fallback response also reported `ollama_chat/llama3.2:1b` in its model field. Both routes still fail if their shared Ollama server becomes unavailable. A production design that needs to survive that outage requires independent inference deployments. Treat token-limit truncation as a response-quality signal and test fallback against the failure classes your service actually intends to handle.

## Follow-on labs

Start these experiments once both model aliases respond through LiteLLM.

| Lab | Architecture concept | Experiment |
| --- | --- | --- |
| 4 | Observability | Compare first-request and repeat-request latency and record token usage when the response includes it. |
| 5 | Access and governance | Explore virtual keys, budgets and model access controls. |
| 6 | Hybrid architecture | Add Amazon Bedrock alongside Ollama and trace which endpoint receives each request. |

[Back to AI, Data and Career Pathways](index.md)
