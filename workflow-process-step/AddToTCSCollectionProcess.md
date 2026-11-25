# AddToTCSCollectionProcess

**Purpose:** add a payload asset to the configured TCS collection, creating the collection if necessary.

**Model:** assign-to-tcs. Its checked-in arguments are parent `/content/dam/collections/tcs` and name `tcs-assets-collection`. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/AddToTCSCollectionProcess.java), [model arguments](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-assign-to-tcs/.content.xml#L23).

## Inputs and flow

1. Resolve the payload resource and require a `jcr:content/metadata` child. The step does not read individual asset metadata properties.
2. Split `PROCESS_ARGS` into collection parent and name; resolve the collection resource.
3. Create the collection with its resource type, title, and description if absent; otherwise get the existing collection. Add the asset and commit.
4. If the collection does not report that it contains the asset, repeat add/commit up to ten times after the initial attempt.

## Outputs and failure behavior

It writes collection membership and possibly the collection node, not asset metadata. It uses AEM collection APIs and makes no external TCS API call.

Missing resolver, asset, metadata, or collection manager returns early. Missing/malformed arguments can cause an unchecked array-index failure. Persistence exceptions are logged; failed membership after retries is logged rather than thrown. Check the actual collection and workflow history.
