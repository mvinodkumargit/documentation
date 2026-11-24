# Belk Assets Cloud — repository guide

This documentation summarizes the checked-out AEM project from source and configuration. It does not claim that a configured workflow, destination collection, endpoint, or permission has been verified in a running AEM environment.

## Start here

- [Project structure and runtime boundaries](project-structure.md)
- [Current flows: launchers, workflows, services, author action, and jobs](flows.md)
- [Detailed product and WIP asset flows](asset-ingestion-flows.md)
- [Workflow catalog and end-to-end maps](workflows/README.md)
- [DAM UI customizations](dam-ui-customizations.md)
- [Product workflows reported in progress and performance review](workflow-in-progress-and-performance.md)
- [Current data and repository model](data-model.md)
- [Technology stack and build](tech-stack.md)
- [Technical gaps and debt](technical-gaps.md)
- [Proposed improvements](proposed-architecture.md)
- [Current architecture diagram](diagrams/current-architecture.mmd)
- [Proposed architecture diagram](diagrams/proposed-architecture.mmd)

## Scope notes

- The codebase is an AEM application built as a Maven multi-module project. The primary runtime behavior is in AEM Author workflows, OSGi services, DAM content configuration, and Dispatcher rules.
- Workflow launchers/models and OSGi configuration are repository declarations. Their effective state can also depend on AEM runtime configuration and administrator-created content.
- The working tree contained pre-existing edits and untracked files during this analysis. The documents describe the inspected working tree; verify the tracked branch and deployed configuration before treating those files as release truth.
- Credentials and token values are intentionally not reproduced here.
