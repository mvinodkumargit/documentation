# Current repository and content model

## Persistence boundary

The application uses AEM's JCR repository and DAM APIs rather than a relational database. Assets are `dam:Asset` resources with metadata below `jcr:content/metadata`; workflows also create/update DAM collections and move or associate assets. No SQL schema, ORM entity mappings, or database transaction manager was found in the declared project structure.

## Main content areas

| Area | Observed role |
|---|---|
| `/content/dam/belk/en/product` | Product assets, including VPI and sample folders; launchers watch rendition and metadata paths beneath this area. |
| `/content/dam/belk/en/wip` | Work-in-progress assets; the asset-purge scheduler defaults to purging expired assets below this root. |
| `/content/dam/belk/en/promotion` and `/content/dam/belk/en/video` | Additional DAM areas with configured author groups and ACLs. |
| `/content/dam/collections` | DAM collection hierarchy used by workflows and author collection operations. |
| `/content/dam/collections/public/tcs/tcs-assets-collection` | Hard-coded target of the TCS author action. The repository code inspected does not prove that this collection is provisioned in the deployed repository. |

Path and ACL creation is configured in the RepoInit script [belk-groups](../ui.config/src/main/content/jcr_root/apps/belkassets/osgiconfig/config.author/org.apache.sling.jcr.repoinit.RepositoryInitializer~belk-groups.cfg.json). The service-user mapping and grants are configured separately in [service-user mapping](../ui.config/src/main/content/jcr_root/apps/belkassets/osgiconfig/config.author/org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-belkconfig.cfg.json) and [service-user RepoInit](../ui.config/src/main/content/jcr_root/apps/belkassets/osgiconfig/config.author/org.apache.sling.jcr.repoinit.RepositoryInitializer~belkassets-service-users.cfg.json).

## Metadata and integration data

- The Belk DAM metadata schema defines fields such as title, description, language, tags/smart tags, and review-related properties: [metadata schema](../ui.content/src/main/content/jcr_root/conf/global/settings/dam/adminui-extension/metadataschema/belk_assets/.content.xml#L7).
- Workflow launchers inspect MIME type, Scene7 publish status, `scene7Url`, and `appliesToAllProducts` to decide whether to start a model. These conditions are listed in [flows.md](flows.md#launcher-rules).
- Salsify integration code maps product properties and updates image URLs/e-commerce status. The exact property identifiers and endpoints are externalized in [Salsify OSGi configuration](../ui.config/src/main/content/jcr_root/apps/belkassets/osgiconfig/config.author/com.belk.aem.assets.core.service.impl.SalsifyApiImpl.cfg.json).
- Scene7 and Workhorse statuses are treated as metadata/integration fields by the workflow steps. The deployed service configuration determines effective endpoints and credentials.
- Workflow payloads and process arguments carry asset paths, workflow identifiers, and collection path/name arguments; these values form implicit contracts between configured workflow models and Java process steps.

## Collection and lifecycle behavior

- Workflows create collections, assign assets, and in some paths move assets into product/sample destinations.
- The TCS UI action adds selected paths to the fixed TCS collection, then attempts to remove the items from the current collection.
- Scheduled maintenance deletes expired DAM assets and empty dated collections. The asset purge also schedules cleanup of associated audit/workflow records.

## Model limitations

This is a repository-level model inventory, not a complete content schema export. Runtime-created assets, metadata fields added by other AEM packages, deployed collection nodes, and external Salsify/Scene7 schemas are not fully discoverable from this repository alone.
