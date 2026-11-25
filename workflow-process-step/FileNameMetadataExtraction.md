# FileNameMetadataExtraction

**Purpose:** validate an asset filename and put its identifiers into workflow metadata for later steps.

**Models:** vendor-provided-images, sample-images, post-processing-sample-vpi, and metadata-asset-workflow. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java), [example model arguments](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L24).

## Inputs and flow

1. Read the model ID, payload, and comma-separated regexes from `PROCESS_ARGS`. Strip the `/jcr:content` suffix when resolving the asset.
2. Require an asset name containing an extension; remove the extension before matching. The checked-in models supply patterns for exactly four or six alphanumeric underscore-separated tokens.
3. On a match, split the name and put the first four tokens in the workflow metadata map:

   | Token | Workflow metadata key | Meaning |
   |---|---|---|
   | 1 | `dam:vendorID` | Vendor identifier |
   | 2 | `dam:styleID` | Style identifier |
   | 3 | `dam:shot` | Shot/order |
   | 4 | `dam:nrfColor` | NRF color |

   Six-token filenames still use only the first four tokens in this mapping. [Mapping helper](../../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L45)

## Writes and integrations

The identifiers are written to the **workflow metadata map**, not directly to `jcr:content/metadata`. The step uses AEM DAM and repository APIs; it makes no external API request. A later [DAM metadata update](DamAssetMetadataUpdate.md) consumes these values.

## Conditions and failure behavior

- A missing extension moves the asset to uncategorized, commits, and terminates the workflow.
- A nonmatching filename normally moves the asset to uncategorized and terminates; for `belk-metadata-asset-workflow`, it terminates without that move.
- The uncategorized helper chooses the Product location only if the payload contains `/product/final-assets`; otherwise it selects WIP uncategorized. [Destination helper](../../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L210)
- Missing resolver, asset manager, or asset causes an early return. Persistence exceptions are logged; a returned step is not proof that the metadata was populated.
