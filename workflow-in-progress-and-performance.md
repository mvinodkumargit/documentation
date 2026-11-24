# Product workflows reported in progress: investigation and performance notes

## Summary

The repository does not establish a single root cause for active workflows. The most direct product-upload trigger is a launcher that starts Adobe's Dynamic Media re-upload model, whose step implementation and runtime behavior are not in this repository. Other product metadata/Scene7 launchers start Belk workflows that make synchronous outbound calls. The HTTP utility does not set request/connect/socket timeouts itself; effective AEM client defaults must be checked.

This is a static source/configuration review. No affected workflow instance IDs, AEM logs, queue metrics, or production timings were supplied, so the hypotheses below require runtime confirmation.

## Product-location trigger map

| Trigger | Configured workflow | Why it matters for "In Progress" |
|---|---|---|
| Modified original rendition at `/content/dam/belk/en/product/(vpi|samples)/.*/renditions/original`; no additional conditions; workflow-process changes excluded | Adobe `/var/workflow/models/dam/dynamic-media-reupload` | This is the most direct match for an uploaded product VPI/sample original. Its target model is Adobe-managed and not defined in this repository, so its active step, timeout, and retry behavior cannot be assessed here. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_product_asset_mod_DM/.content.xml#L5) |
| Modified product metadata node where `scene7Url` is nonempty and `appliesToAllProducts == true` | Belk asset-metadata-save; one Salsify update step | The process synchronously calls Salsify before terminating. A slow or non-returning remote call could delay workflow completion; no live instance/log evidence confirms this is happening. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-asset-metadata-save/.content.xml#L5), [model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-asset-metadata-save/.content.xml#L23), [process](../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyOnAssetSaveProcess.java#L89) |
| Product VPI/sample metadata reaches `dam:scene7FileStatus == PublishComplete`, with configured property-change exclusions | Belk post-Scene7 model: swatch extraction → Salsify update → CDN invalidation → Workhorse status | Steps are sequential. A slow outbound operation can hold the workflow at its current step; the exact active instance and response latency are not available. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-post-scene7-url-to-salisfy/.content.xml#L5), [model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-scene7-urls-to-salsify/.content.xml#L23) |

Other routing is performed by `InvokeWorkflowModel`, which selects models using the workflow ID and payload asset path; see [InvokeWorkflowModel.java](../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L67). The complete launcher/model inventory is in [flows.md](flows.md#launcher-rules).

## Evidence and candidate causes

### Confirmed by source

- The product rendition launcher starts an external Adobe Dynamic Media workflow. Its steps and completion conditions are outside this repository. [Launcher config](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_product_asset_mod_DM/.content.xml#L5)
- The metadata-save workflow calls `SalsifyApi.updateImageURLforNRFAtSalsify(...)` synchronously, then terminates its workflow. [SalsifyOnAssetSaveProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyOnAssetSaveProcess.java#L89)
- The post-Scene7 model runs its Salsify, CDN, and Workhorse steps sequentially. The Salsify update calls `salsifyApi.updateDataAtSalsify(propertyMap)` synchronously. [SalsifyDataUpdateProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyDataUpdateProcess.java#L44)
- `RestClient` executes GET/POST/PUT requests without setting request, connect, or socket timeouts in those methods. The underlying AEM HTTP client factory may impose defaults; verify effective configuration before concluding calls are unbounded. [RestClient.java](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L99)
- The CDN invalidation step catches and logs exceptions. This confirms its local failure handling, but does not establish whether an instance is slow or stuck. [Scene7CDNCacheInvalidation.java](../core/src/main/java/com/belk/aem/assets/core/workflows/Scene7CDNCacheInvalidation.java#L95)

### Plausible but unconfirmed

- A Dynamic Media external process may be slow or waiting on an external service. Confirm the instance's active step and Adobe Dynamic Media status/logs.
- A Salsify or CDN operation may take a long time if the remote service is slow and the effective HTTP client timeout is high or absent.
- Multiple workflow instances could be created for one asset if multiple matching repository events occur. The code review alone does not establish duplicate triggering; compare instance payloads, timestamps, and launcher event details.
- Errors caught and logged by a process do not by themselves explain an instance staying `In Progress`. Inspect the workflow history and current step rather than inferring the cause from a log message.

## Performance findings

These are source-visible risk areas. Their actual production impact depends on traffic volume, index state, AEM client defaults, external service latency, and repository size.

| Area | Source evidence | Possible impact |
|---|---|---|
| Outbound HTTP | GET/POST/PUT execute without timeouts set in `RestClient`. [GET](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L99), [POST](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L146), [PUT](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L219) | Slow downstream calls can occupy workflow/job threads longer than expected, depending on effective HTTP-client defaults. |
| Salsify product lookup | Up to three sequential fallback GET requests; each requests up to 250 results. [Fallback chain](../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L207), [page size](../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L279) | Repeated remote latency and response parsing can accumulate when earlier lookups return no products. |
| Asset purge | Query limit is `-1`, then every result and its versions are iterated; commits are batched by 50. [Query/iteration](../core/src/main/java/com/belk/aem/assets/core/schedulers/AssetPurgeScheduler.java#L201), [version iteration](../core/src/main/java/com/belk/aem/assets/core/schedulers/AssetPurgeScheduler.java#L270) | Large expiry backlogs or version histories may produce long runs and repository/resource pressure. This job is not itself a product-upload workflow. |
| Multi-search predicate | Builds one query predicate per submitted code, iterates all query results/metadata, and logs inputs/results at INFO. [Predicate construction](../core/src/main/java/com/belk/aem/assets/core/search/eval/MultiSearchPredicateEvaluator.java#L120), [result iteration](../core/src/main/java/com/belk/aem/assets/core/search/eval/MultiSearchPredicateEvaluator.java#L216) | Large code lists/results can amplify query work and log volume. This is a DAM search cost, not a direct cause of an upload workflow remaining active. |

The observations above do not prove an incident or quantify latency. Baseline step durations and query/result counts are needed to prioritize them.

## Runtime triage checklist

For each affected asset, capture a workflow instance ID, asset payload path, model name, start time, and current step.

1. **Identify the matching launcher.** An original under product `vpi`/`samples` points to Adobe Dynamic Media re-upload; metadata conditions may point to asset-metadata-save or post-Scene7. Compare the instance's model and payload against the launcher table above.
2. **Inspect workflow history and active step.** Check whether the instance is actively executing, waiting for an external process, retrying, or stalled in a queue. Do not infer this from the `In Progress` label alone.
3. **For Dynamic Media:** inspect the external model's current step, Dynamic Media processing status, and corresponding AEM logs. The model is not defined in this repository.
4. **For Salsify/Scene7/Workhorse:** correlate workflow timestamps with outbound-call duration, status, exception, and remote service logs. Confirm effective HTTP timeout configuration; never log raw tokens or payload secrets.
5. **Check for duplicate starts.** Compare same-asset workflow instances and their event paths/properties; verify whether workflow-generated metadata changes satisfy launcher conditions.
6. **Check queue and resource pressure.** Compare active/queued workflow counts, job queue backlog, thread utilization, and repository/query latency during the affected window.

## Recommended follow-up improvements

- Set explicit connection and response timeouts for outbound HTTP and surface a typed success/failure outcome to workflow callers.
- Add per-step start/end duration, outcome, and correlation ID telemetry; use redacted values and aggregate metrics.
- Define bounded retry/backoff and idempotency per external write; make terminal failures visible in the workflow state and operator report.
- Review launcher conditions/exclusions against real event traces to prevent duplicate work without suppressing required updates.
- Paginate or otherwise bound purge queries and report processed/deleted/skipped/failed counts.
- Bound multi-search input/result work and reduce per-result INFO logging.

These recommendations are additive to the broader items in [technical-gaps.md](technical-gaps.md).
