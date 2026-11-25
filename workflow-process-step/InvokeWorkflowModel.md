# InvokeWorkflowModel

**Purpose:** start other workflow instances based on the current model and payload.

**Configured model:** batch-thumbnails-post-to-salsify-call. The other code branches below are not shown wired to this process in the checked-in model definitions. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java), [configured step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-batch-thumbnails-post-to-salsify-call/.content.xml#L23).

## Branches in execution order

1. **Batch-thumbnails model:** resolve the payload resource and read its multi-value `paths` property. For each entry containing `/product/vpi/` or `/product/samples/`, start a `belk-post-scene7-urls-to-salsify` child workflow with that entry as the JCR-path payload. Return after this branch.
2. **Post-processing model:** read workflow metadata `asset-path`; if nonempty, start `belk-scene7-batch-upload` with that path. Otherwise log and skip.
3. **Other model IDs:** a payload containing `/product/final-assets` starts `belk-post-processing-sample-vpi`; otherwise `/wip/samples` starts `belk-sample-images`; all other paths start `belk-vendor-provided-images`.

## Outputs and caveats

The side effect is starting child workflow instances. The batch branch reads a repository property; it does not persist asset metadata or directly call Salsify/Scene7. Starting a child does not prove the child or its external requests completed.

The checked-in batch model reaches branch 1 and returns before general routing. Neither a normal trigger of branches 2–3 nor an automatic WIP-to-Product transition is established by the repository. Missing resolver or `paths` skips work; missing resources, invalid values, or workflow-engine exceptions can fail the step.
