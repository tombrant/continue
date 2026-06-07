---
title: Azure OpenAI 429 Mitigation Recommendation
description: Recommended strategy for reducing Azure OpenAI 429 errors in Continue by combining payload pruning, retry behavior, and request shaping.
---

# Azure OpenAI 429 Mitigation Recommendation

## Summary

Payload pruning reduced request body size significantly in testing, but it did **not** eliminate `429 Too Many Requests` responses.

That result is expected. For Azure OpenAI, `429` is not caused only by large request bodies. It can also be triggered by:

- Tokens-per-minute (TPM) or requests-per-minute (RPM) quota exhaustion
- Temporary backend scaling delays
- Sharp spikes in request volume
- Large `max_tokens` or similar output-token settings
- Multiple concurrent in-flight requests against the same deployment

The recommended fix is therefore **not** "prune until below a hard byte limit and assume success." The recommended fix is a layered mitigation strategy.

## Final Recommendation

Use a combined approach:

1. **Prune proactively before sending**
2. **Prune to a conservative target, not just below a hard ceiling**
3. **Retry on 429 with exponential backoff**
4. **Retry once with more aggressive pruning**
5. **Reduce request cost drivers such as high output-token settings**
6. **Reduce burstiness and unnecessary concurrency**
7. **Measure both bytes and estimated tokens**

## Why payload pruning alone is insufficient

Observed behavior showed requests trimmed below a local threshold still receiving `429` responses.

This indicates that a local byte budget is only a proxy and not the actual admission rule used by Azure OpenAI.

Azure guidance aligns with this interpretation:

- Azure may return `429` when workload exceeds quota
- Azure may also return `429` during backend scale-up, even if the application appears to be within expected limits
- Large `max_tokens` and related parameters contribute to quota pressure
- Retry with backoff is explicitly recommended

## Recommended request strategy

### 1. Keep proactive pruning

Continue using pruning before sending requests because it materially reduces payload cost.

Recommended pruning order:

- Remove inline `file_data` where `file_id` is available
- Remove or reduce older tool-related turns first
- Remove oldest complete conversation turns
- Truncate or summarize older verbose content
- Preserve system/developer messages and recent user intent

### 2. Prune to a safety margin

Do not prune only until the body falls just under a configured maximum such as `200000` bytes.

Instead, define:

- A **hard limit**
- A lower **target limit**

Recommended defaults:

- Hard limit: `200000` bytes
- First-attempt target: `160000` bytes
- Retry target after 429: `120000` to `130000` bytes

This creates clearance for Azure-side variability.

### 3. Add retry-on-429 behavior

Implement exponential backoff with jitter.

Recommended policy:

- First attempt: send normally with conservative pruning
- If Azure returns `429`:
  - wait with backoff
  - rebuild or re-prune the payload more aggressively
  - retry once
- If the second attempt also returns `429`, fail with a specific diagnostic

This matches Azure guidance and avoids over-pruning every request up front.

### 4. Prefer token-aware pruning, not byte-only pruning

Byte size is useful, but Azure throttling is more likely tied to token and request budgets than raw payload bytes.

Use both:

- JSON body byte count
- Estimated prompt tokens

If available in the codebase, use an existing tokenizer. Otherwise, use a rough estimate as a fallback.

Recommended trigger:

- prune if bytes exceed target budget
- or estimated prompt tokens exceed target prompt-token budget

### 5. Reduce output-token pressure

Large output settings can increase quota pressure.

Review and reduce where possible:

- `max_tokens`
- `max_completion_tokens`
- `best_of`

Use only the amount needed for the request.

### 6. Reduce concurrency and burstiness

If multiple requests hit the same Azure deployment at once, `429` becomes more likely even when individual requests are moderate in size.

Recommended controls:

- debounce repeated background requests
- avoid duplicate overlapping requests for the same interaction
- optionally serialize or cap concurrent requests per deployment
- ramp up workload gradually rather than in bursts

## Recommended pruning behavior

## Conversation integrity requirements

Pruning must preserve valid chat structure.

In particular:

- A `tool` message must not remain without its preceding assistant `tool_calls`
- Tool-related message chains should be pruned as a unit
- Prefer pruning complete turns rather than individual messages

Recommended preservation priorities:

- keep all system/developer messages
- keep the most recent user turn
- keep the most recent assistant/tool block if relevant
- drop older tool-heavy turns before dropping newer plain-chat turns

## Recommended defaults

<Card title="Suggested Defaults" icon="gauge">

  Use these values as a starting point:
  - `payloadMaxBytes`: `200000`
  - `pruneTargetRatio`: `0.8`
  - `retryPruneTargetRatio`: `0.65`
  - retry count for `429`: `1`
  - retry strategy: exponential backoff with jitter

</Card>

This implies:

- first attempt target: `160000` bytes
- retry attempt target: `130000` bytes

For heavier deployments such as `gpt-5.4`, consider even lower first-attempt targets.

## Implementation recommendation

The preferred implementation sequence is:

1. Keep the current pre-send pruning hook in request construction
2. Change it from "prune to hard limit" to "prune to target ratio"
3. Add tool-aware pruning so message structure remains valid
4. Add retry-on-429 in the Azure/OpenAI request path
5. On retry, apply stricter pruning and optionally reduce output-token settings
6. Add metrics for bytes, estimated tokens, message count, tool-message count, and retry attempt number

## Diagnostics to log

Add structured logs for each request:

- original body bytes
- final body bytes
- estimated prompt tokens
- message count before and after pruning
- tool-related message count before and after pruning
- configured hard limit
- configured target limit
- retry attempt number
- model and deployment name

This will make it easier to determine whether failures correlate most strongly with:

- prompt size
- output-token settings
- concurrency
- specific deployments
- traffic bursts

## What not to assume

Do not assume any of the following:

- that staying below a fixed byte threshold guarantees success
- that all `429` responses are caused by prompt size
- that pruning alone is sufficient
- that the same threshold will work consistently across all Azure deployments or times of day

## Final conclusion

The best mitigation is a **combined strategy**:

<CardGroup>
  <Card title="Before Send" icon="scissors">

    Prune proactively and preserve a safety margin:
    - remove high-cost payload elements
    - preserve recent and structurally valid context
    - target a budget below the configured maximum

  </Card>

  <Card title="On 429" icon="rotate-right">

    Retry once with stronger controls:
    - exponential backoff with jitter
    - more aggressive pruning
    - optionally lower output-token settings

  </Card>

  <Card title="Operational Controls" icon="chart-line">

    Reduce upstream pressure:
    - smooth bursts
    - limit concurrency
    - monitor TPM/RPM usage
    - tune deployment quotas where possible

  </Card>
</CardGroup>

This approach is more robust than either of these simpler strategies alone:

- pruning only after a `429`
- pruning only to a local maximum byte threshold and assuming success

## Azure-specific considerations

<Info>

  Azure OpenAI `429` responses can happen for more than one reason:
  - defined TPM/RPM limits may be exceeded
  - the service may be scaling to demand and temporarily unable to serve the request
  - large `max_tokens` or `best_of` settings can increase rate-limit pressure
  - retry with backoff is recommended by Azure guidance

</Info>

## Action items

- Update pruning to use a target ratio below the hard ceiling
- Make pruning tool-chain aware
- Add retry-on-429 with exponential backoff and jitter
- Rebuild requests with stricter pruning on retry
- Audit and lower high output-token defaults where possible
- Add request concurrency controls for Azure deployments
- Add request cost telemetry for bytes and estimated tokens