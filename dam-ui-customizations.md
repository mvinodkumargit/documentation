# DAM UI customizations

This document catalogs the repository-defined customizations that affect the AEM DAM authoring UI. Adobe-provided `/libs` components and runtime-installed behavior are identified as dependencies, not described as Belk-owned code.

## Overview

| Surface | Repository customization | User-visible effect |
|---|---|---|
| Collection details | Extends the Adobe collection-details page and adds a selection action plus timeline configuration. | Adds “Move To TCS Collection”; sets a multiple-workflow limit for the inherited timeline. |
| Action client library | Adds a client-side listener for the custom collection action. | Adds selected assets to a fixed TCS collection and then removes them from the current collection. |
| Main-asset render condition | Implements the `dam/gui/content/mainasset` condition used by the action. | Hides the action for subassets/pages and for a path containing the TCS collection name. |
| Asset metadata editor | Defines the Belk DAM metadata schema. | Controls configured metadata tabs/fields, including smart-tag and review-related properties. |
| DAM search rail | Defines the Assets Admin Search Rail facets and property predicates. | Exposes filtering/search controls for asset location, type, status, and Belk-specific identifiers. |
| Search predicate | Core bundle registers a custom `multisearch` QueryBuilder predicate. | Supports searches for multiple metadata values using a constrained property list. |

## Collection-details action

The repository adds content below the DAM collection-details page; the page declares the Adobe collection-details page as its parent. The action is named **“Move To TCS Collection”**, targets selected child pages, and is guarded by the custom `dam/gui/content/mainasset` render condition. The inherited page/timeline behavior remains Adobe-provided. [Collection details configuration](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/collections/collectiondetails/.content.xml#L4)

The `tcscollection` client library is registered in the `dam.gui.actions.coral` category and lists dependencies on Adobe DAM action/viewer and admin utility libraries. [Client library definition](../ui.apps/src/main/content/jcr_root/apps/belkassets/clientlibs/tcscollection/.content.xml#L1), [client library manifest](../ui.apps/src/main/content/jcr_root/apps/belkassets/clientlibs/tcscollection/js.txt#L1)

### Action behavior

1. The script listens for the action activation and collects selected asset paths; it also supports a quick-action path.
2. It posts an AEM collection `:operation=add` request to `/content/dam/collections/public/tcs/tcs-assets-collection.collection.html`.
3. If adding succeeds, it attempts to remove the selected assets from the current collection and displays a success dialog.
4. It has user-facing error handling for permission/cycle cases and failures.

The request is synchronous (`async: false`), and the add and remove are separate operations. Thus an add can succeed while the removal fails. The script does not use a custom application endpoint; it invokes AEM collection operations directly. [Action JavaScript](../ui.apps/src/main/content/jcr_root/apps/belkassets/clientlibs/tcscollection/js/addtotcscollection.js#L19)

## Main-asset eligibility

The action's render condition resolves either a configured resource path or the current suffix resource. It excludes subassets/pages beneath a `subassets` folder, and also excludes any resource path containing `tcs-assets-collection`. [Render condition implementation](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/mainasset/mainasset.jsp#L29)

The implementation is repository-defined; the Sling render-condition API and surrounding Coral/Granite authoring UI are Adobe-provided. The check is based on a path substring rather than a configured collection identity. The code also assumes adaptation of the resolved resource to a JCR `Node` succeeds.

## Asset metadata schema

The configured schema is at `conf/global/settings/dam/adminui-extension/metadataschema/belk_assets`. It defines a Coral tabs schema with a **Basic** tab and fields including:

- Title and Description
- Type (disabled in the configured form)
- Language
- Tags and Smart Tags
- Created and Creator Tool
- Review Status

Smart Tags visibility uses Adobe's autotag render condition. The field/form/schema resource types are Adobe components; the Belk repository defines the schema node structure, field properties, labels, and ordering. [Metadata schema](../ui.content/src/main/content/jcr_root/conf/global/settings/dam/adminui-extension/metadataschema/belk_assets/.content.xml#L7)

## Asset search rail

The repository defines the **Assets Admin Search Rail** in `conf/global/settings/dam/search/facets/assets`. Configured controls cover:

- Path and search type
- File type, file size, and last modified
- Status facets for publish, approval, checkout, checked-out-by, and expiry
- Orientation
- Belk-specific multi-value search properties: Studio Set ID, Brand, PIM NRF Color Code, and Belk Style ID

Belk defines the facet configuration, labels, order, and searched properties. Granite predicate resource types and option sources under `/libs` are Adobe-provided. [Search facet configuration](../ui.content/src/main/content/jcr_root/conf/global/settings/dam/search/facets/assets/.content.xml#L4)

The custom `multisearch` predicate is implemented in [MultiSearchPredicateEvaluator.java](../core/src/main/java/com/belk/aem/assets/core/search/eval/MultiSearchPredicateEvaluator.java#L47). It parses path/property/code input, allows only a fixed set of metadata properties, checks existing values through QueryBuilder, may create a low-priority task when codes are missing, and returns an XPath restriction. Search-cost and logging concerns are covered in [workflow and performance notes](workflow-in-progress-and-performance.md#performance-findings).

## Other DAM-related repository configuration

- The `/apps/dam` namespace and collection-details nodes provide the repository overlay path; their node definitions do not themselves add a dialog or action. [DAM namespace node](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/.content.xml#L1), [collections node](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/collections/.content.xml#L1)
- The repository includes `conf/belkassets` and DAM settings containers, but the inspected nodes do not define additional custom authoring dialogs/actions. [Belk configuration root](../ui.content/src/main/content/jcr_root/conf/belkassets/.content.xml#L1), [DAM settings root](../ui.content/src/main/content/jcr_root/conf/belkassets/settings/dam/.content.xml#L1)
- No custom Granite dialog definition or action-specific server-side endpoint was found in the inspected DAM paths. Adobe defaults and runtime-installed `/libs` behavior are outside the repository and cannot be fully inventoried here.

## Follow-up concerns

- The TCS destination path is hard-coded. No matching collection provisioning path was found in the inspected packaged content, so existence in each target environment is unverified.
- The action can block the authoring browser while its synchronous request runs, and its two-stage update can leave partial state.
- The search rail configuration uses older `granite/ui/components/foundation` resource types in places while other custom UI uses Coral types. This is a consistency/maintenance concern, not proof of incompatibility.
