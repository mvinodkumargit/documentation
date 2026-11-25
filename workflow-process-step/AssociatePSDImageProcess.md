# AssociatePSDImageProcess

**Purpose:** find matching WIP assets, add DAM source relations, and make the result available to the following metadata-copy step.

**Model:** post-processing-sample-vpi. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/AssociatePSDImageProcess.java).

## Inputs and flow

1. Read workflow metadata `asset-path` from the preceding move step; skip if missing or empty.
2. Read the moved asset's `nrfColorCode`, `vendorId`, and `styleId` as string arrays, plus `shotOrder` as a string. The first array element is used for each query value.
3. Query `dam:Asset` resources under `/content/dam/belk/en/wip` matching the four metadata properties. The query has `p.limit=-1`.
4. Read the moved asset's existing `sources` relations and add one for each query hit not already related. Commit, then place the list of query `Hit` objects in workflow metadata as `assetList`.

Despite the process name, the query shown in the source does **not** filter by PSD/TIFF extension or MIME type. [Query construction](../../core/src/main/java/com/belk/aem/assets/core/workflows/AssociatePSDImageProcess.java#L111)

## Outputs, integrations, and errors

It writes DAM asset relations and workflow metadata, not asset metadata properties; the next [metadata-copy step](UpdateRelatedAssetSalsifyMetadata.md) reads `assetList`. It uses AEM Query Builder and DAM APIs, with no external HTTP call.

Missing path or metadata causes a skip. Empty arrays, missing asset resources, or null asset objects can cause runtime failures. Persistence and repository exceptions around the query/relation operations are logged. The unbounded query can be expensive for many matching WIP assets.
