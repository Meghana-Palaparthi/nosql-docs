---
ms.topic: include
ms.date: 04/29/2026
---
### Customize Azure OpenAI deployment (optional)

The infrastructure deploys Azure OpenAI with default model and region settings defined in `infra/main.bicepparam`. Before running `azd up`, you can customize any of these parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AZURE_OPENAI_LOCATION` | Same as `AZURE_LOCATION` | Region for OpenAI resource deployment |
| `AZURE_OPENAI_CHAT_MODEL` | `gpt-4.1-mini` | Chat completion model name |
| `AZURE_OPENAI_CHAT_MODEL_VERSION` | `2025-04-14` | Chat model version |
| `AZURE_OPENAI_CHAT_MODEL_TYPE` | `Standard` | Deployment type (`Standard` or `GlobalStandard`) |
| `AZURE_OPENAI_SYNTH_MODEL` | `gpt-4.1` | Synthesis/reasoning model name |
| `AZURE_OPENAI_SYNTH_MODEL_VERSION` | `2025-04-14` | Synthesis model version |
| `AZURE_OPENAI_SYNTH_MODEL_TYPE` | `Standard` | Deployment type (`Standard` or `GlobalStandard`) |
| `AZURE_OPENAI_EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model name |
| `AZURE_OPENAI_EMBEDDING_MODEL_VERSION` | `1` | Embedding model version |
| `AZURE_OPENAI_EMBEDDING_MODEL_TYPE` | `Standard` | Deployment type (`Standard` or `GlobalStandard`) |

To override a default, use `azd env set` before running `azd up`:

```bash
# Deploy OpenAI to a different region than your other resources
azd env set AZURE_OPENAI_LOCATION swedencentral

# Switch deployment type from Standard (default) to GlobalStandard
azd env set AZURE_OPENAI_CHAT_MODEL_TYPE GlobalStandard

# Change the chat model
azd env set AZURE_OPENAI_CHAT_MODEL gpt-4.1-mini
azd env set AZURE_OPENAI_CHAT_MODEL_VERSION 2025-04-14
```

> [!NOTE]
> Not all models are available in all regions or with all deployment types. Check [Azure OpenAI model availability by region](/azure/ai-services/openai/concepts/models#model-summary-table-and-region-availability) for supported combinations.

### Troubleshoot Azure OpenAI provisioning failures

If `azd up` fails when creating the Azure OpenAI resource or model deployments, the problem is typically one of:

- **Region availability**: The model isn't available in your chosen region. Try `azd env set AZURE_OPENAI_LOCATION <different-region>` (for example, `eastus2` or `swedencentral`).
- **Deployment type mismatch**: The model doesn't support the selected deployment type in your region. Switch between `Standard` and `GlobalStandard` using `azd env set AZURE_OPENAI_CHAT_MODEL_TYPE GlobalStandard`.
- **Quota limits**: Your subscription reached its quota for the selected model, region, or deployment type combination. Check your quota in the Azure portal under **Azure OpenAI** > **Quotas**. You can request a quota increase or try a different region where you have available capacity.
- **Model retired or unavailable**: Azure OpenAI periodically retires older model versions. If deployment fails because a model version is no longer available, update to a supported version using `azd env set AZURE_OPENAI_CHAT_MODEL_VERSION <new-version>` (or the equivalent for embedding/synth models). Check [Azure OpenAI model retirements](/azure/ai-services/openai/concepts/model-retirements) for current model lifecycle status.

After changing any parameters, run `azd up` again to retry the deployment.