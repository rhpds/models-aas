# Llama 4 Scout on Intel Gaudi3

Deployment for Meta Llama 4 Scout 17B (16 experts) on Intel Gaudi3 accelerators.

## Model Specifications

- **Model:** meta-llama/Llama-4-Scout-17B-16E-Instruct
- **Parameters:** 17B activated / 109B total (MoE with 16 experts)
- **Context:** 10M tokens (deployed with 512K, scalable)
- **Multimodal:** Text + Image → Text + Code
- **Function Calling:** Supported
- **Max Images:** 5 per request
- **License:** Llama 4 Community License (gated)

## Hardware

- **Accelerators:** 4x Intel Gaudi3 (out of 8 available)
- **Memory:** 128Gi
- **Storage:** 250Gi LVMS (lvms-vg1)
- **Node:** maas00-sno

## Prerequisites

1. Intel Gaudi Operator installed
2. Node labeled: `accelerator=gaudi`
3. HuggingFace token secret in llm-hosting namespace
4. Model access approved on HuggingFace

## Deployment

```bash
# Deploy
oc apply -f deployment.yaml

# Monitor
oc get pods -n llm-hosting -l app=llama-4-scout -w
oc logs -f -n llm-hosting -l app=llama-4-scout

# Get route
oc get route llama-4-scout -n llm-hosting
```

## Testing

```bash
ROUTE=$(oc get route llama-4-scout -n llm-hosting -o jsonpath='{.spec.host}')

# Health check
curl https://$ROUTE/health

# Chat completion
curl https://$ROUTE/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-4-Scout-17B-16E-Instruct",
    "messages": [{"role": "user", "content": "What is OpenShift?"}],
    "max_tokens": 150
  }'

# Vision (image understanding)
curl https://$ROUTE/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-4-Scout-17B-16E-Instruct",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
        {"type": "text", "text": "Describe this image"}
      ]
    }]
  }'

# Function calling
curl https://$ROUTE/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-4-Scout-17B-16E-Instruct",
    "messages": [{"role": "user", "content": "Get weather in Boston"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather for a city",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string"}
          },
          "required": ["location"]
        }
      }
    }],
    "tool_choice": "auto"
  }'
```

## Integration with LiteMaaS

Add via LiteMaaS Admin UI:

1. Login: https://litellm-prod-frontend.apps.maas.redhatworkshops.io/login
2. Navigate to: **Models** → **Add Model**
3. Configure:
   - **Model Name:** `llama-4-scout-17b-gaudi`
   - **LiteLLM Model:** `openai/meta-llama/Llama-4-Scout-17B-16E-Instruct`
   - **API Base:** `https://<route>/v1`
   - **API Key:** `sk-none` (no auth)
   - **RPM:** `30`
   - **TPM:** `50000`
   - **Max Tokens:** `524288`
   - **Supports Vision:** ✓ Yes
   - **Supports Function Calling:** ✓ Yes
4. Sync models:
   ```bash
   cd ~/rhpds.litemaas
   ./sync-models.sh litellm-rhpds
   ```

## Scaling Context Length

Start with 512K, scale up gradually:

```bash
# Test 1M tokens
oc set env deployment/llama-4-scout-tgi -n llm-hosting \
  MAX_INPUT_LENGTH=1048576 \
  MAX_TOTAL_TOKENS=1048576

# Test 5M tokens
oc set env deployment/llama-4-scout-tgi -n llm-hosting \
  MAX_INPUT_LENGTH=5242880 \
  MAX_TOTAL_TOKENS=5242880

# Full 10M tokens (if memory allows)
oc set env deployment/llama-4-scout-tgi -n llm-hosting \
  MAX_INPUT_LENGTH=10485760 \
  MAX_TOTAL_TOKENS=10485760
```

## Troubleshooting

**Pod stuck in Pending:**
```bash
oc describe pod -n llm-hosting -l app=llama-4-scout
```

**Model download fails:**
```bash
# Verify HF token
oc get secret hf-token -n llm-hosting -o jsonpath='{.data.token}' | base64 -d
```

**Out of memory:**
```bash
# Reduce to 3 Gaudi cards
oc set env deployment/llama-4-scout-tgi -n llm-hosting NUM_SHARD=3
```

## Maintenance

**Update TGI-Gaudi image:**
```bash
oc set image deployment/llama-4-scout-tgi -n llm-hosting \
  tgi-gaudi=ghcr.io/huggingface/tgi-gaudi:latest
```

**Remove deployment:**
```bash
oc delete -f deployment.yaml
```
