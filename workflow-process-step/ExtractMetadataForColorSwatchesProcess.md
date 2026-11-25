# ExtractMetadataForColorSwatchesProcess

**Purpose:** construct Scene7 URL properties on the DAM asset for later downstream steps.

**Model:** post-scene7-urls-to-salsify, before the Salsify URL update. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/ExtractMetadataForColorSwatchesProcess.java), [model](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-scene7-urls-to-salsify/.content.xml#L20).

> This page describes the current working tree, including local uncommitted changes to this class. Deployed behavior can differ.

## Inputs and flow

1. Read the payload; if it ends in `jcr:content/metadata`, strip that suffix and resolve the asset.
2. Read DAM metadata `dam:scene7Domain`, `dam:scene7File`, and `dam:crop`.
3. If both domain and file are nonempty, use the configured Scene7 and swatch URL parameters and viewer settings provided by `SalsifyApi` to build:
   - a main image URL using the domain, file folder/name, layer and `src`;
   - a swatch URL including the `dam:crop` value;
   - a viewer URL containing the viewer, server, configuration, content, and asset parameters.
4. Write `scene7Url` and `scene7ViewerUrl`. Write `scene7SwatchUrl` only when `dam:crop` is nonempty. The step attempts to commit the metadata.

The swatch URL is constructed even when crop is absent, so its URL-parameter configuration is still accessed. An absent crop does not clear a pre-existing `scene7SwatchUrl`. The repository does not establish how `dam:crop` is generated.

## Integrations and failure behavior

This step reads local OSGi-backed settings through `SalsifyApi`; it does **not** make a Salsify or Scene7 HTTP request. The following [Salsify URL step](SalsifyDataUpdateProcess.md) uses its output.

In the current working tree, an unresolved asset throws a `WorkflowException`; missing/too-short URL setting arrays become a `WorkflowException` naming the setting. The guard checks array presence and length, not whether entries are blank. Missing resolver or metadata returns early; missing domain/file results in no URL updates. Persistence exceptions are logged, and the shared metadata helper can log rather than propagate individual write/commit failures.
