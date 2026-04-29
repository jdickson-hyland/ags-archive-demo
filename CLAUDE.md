# Alfresco AGS Filesystem Archive Extension

## Project Overview

This is an **Alfresco In-Process SDK 4.13** project targeting **Alfresco Enterprise 25.x**.
It implements a custom AGS disposition action that archives records (content + metadata + audit)
to a local filesystem when an accession event fires in the Alfresco Governance Services lifecycle.

## Key Commands

```bash
# Build the project (skipping integration tests)
mvn clean package -DskipTests

# Run with embedded Docker (starts ACS + Share + Search + Postgres)
mvn clean install -Prun

# Run integration tests against a running instance
mvn integration-test

# Stop and clean Docker containers
./run.sh stop
./run.sh purge
```

## Project Structure

```
src/
  main/
    java/com/yourcompany/archive/
      action/          # Custom disposition action (RMActionExecuterAbstractBase)
      service/         # ArchiveService — content + metadata + audit extraction
      listener/        # TransactionListenerAdapter for safe filesystem writes
      model/           # ArchiveManifest, MetadataExport POJOs
    resources/
      alfresco/
        module/        # module.properties, context XML files
        extension/     # Spring bean wiring XMLs
  test/
    java/              # Unit and integration tests
```

## Architecture: How the Archiving Works

### Trigger
A custom `RMActionExecuterAbstractBase` subclass is registered as a disposition action.
It fires when AGS executes an **accession** step in the disposition schedule.
It is wired via `dispositionActionExecutors` bean in Spring XML.

### Transaction Safety
All filesystem writes happen in an `afterCommit()` TransactionListenerAdapter.
Never write to disk inside the main transaction body — the accession may roll back.

```java
AlfrescoTransactionSupport.bindListener(new TransactionListenerAdapter() {
    @Override public void afterCommit() { archiveService.writeToFilesystem(nodeRef, targetDir); }
});
```

### Output Layout
Archive root is configured via `alfresco-global.properties`:
```
archive.filesystem.root=/opt/alfresco/archive
```

Directory structure mirrors the AGS File Plan hierarchy:
```
{archive.filesystem.root}/
  {record-category}/
    {record-folder}/
      {record-identifier}/
          content.{ext}       # Binary content streamed from ContentService
          metadata.json       # All node properties (QName -> value)
          disposition.json    # Retention schedule and accession details
          audit.json          # AuditService snapshot for this node
```

### Key Alfresco APIs Used

| Task | API |
|------|-----|
| Read node properties | `NodeService.getProperties(nodeRef)` |
| Stream file content | `ContentService.getReader(nodeRef, ContentModel.PROP_CONTENT)` |
| Get AGS record identity | `RecordsManagementModel.PROP_IDENTIFIER` |
| Read audit entries | `AuditService.auditQuery(callback, params, max)` |
| Register disposition action | `RecordsManagementActionService` + Spring XML |
| React to property changes | `PolicyComponent.bindClassBehaviour(OnUpdatePropertiesPolicy...)` |

## Coding Conventions

- Java 17 (required by SDK 4.13 / ACS 25.x)
- Spring XML bean wiring (no annotation-based Spring — Alfresco convention)
- Package root: `com.yourcompany.archive`
- Content streamed — never load binary into a byte array
- Check for existing archive path before writing (idempotent)
- Write a status property back to the node after successful archive (`archive:status`, `archive:path`)

## Important AGS Model References

```java
// AGS namespace
RecordsManagementModel.RM_URI          // "http://www.alfresco.org/model/recordsmanagement/1.0"

// Key properties
RecordsManagementModel.PROP_IDENTIFIER          // rma:identifier
RecordsManagementModel.PROP_DISPOSITION_INSTRUCTIONS
RecordsManagementModel.ASPECT_RECORD
RecordsManagementModel.TYPE_FILE_PLAN
```

## Configuration (alfresco-global.properties)

```properties
# Filesystem archive root — must be writable by the Alfresco process user
archive.filesystem.root=/opt/alfresco/archive

# Whether to write an audit snapshot alongside content and metadata
archive.include.audit=true

# Whether to fail the accession if the filesystem write fails
archive.failOnError=true
```

## Testing Approach

1. **Unit tests** — mock `NodeService`, `ContentService`, `AuditService` with Mockito
2. **Integration tests** — use Alfresco SDK embedded Docker; create a test File Plan,
   declare a record, trigger accession, assert files exist at expected path
3. **Verify idempotency** — run accession trigger twice; confirm no duplicate or corrupted output

## Enterprise Repository Access

Enterprise artifacts require credentials in `~/.m2/settings.xml`:

```xml
<server>
  <id>alfresco-private-repository</id>
  <username>YOUR_ENTERPRISE_USERNAME</username>
  <password>YOUR_ENTERPRISE_PASSWORD</password>
</server>
```

The Alfresco private repository URL is:
`https://artifacts.alfresco.com/nexus/content/groups/private/`

## Known Constraints and Watch-outs

- Do **not** use `@Component` / `@Service` Spring annotations — use explicit XML bean definitions
  in `src/main/resources/alfresco/extension/`. Alfresco's Spring context does not scan annotations
  in the same way as a standard Spring Boot app.
- AGS JARs (`alfresco-rm-community-repo` or enterprise equivalent) must be added as `provided`
  scope dependencies — they are supplied by the running ACS instance.
- `RecordsManagementActionService` bean name in XML is `rmActionService`.
- When registering a new disposition action, the action name string must be unique and match
  exactly between the Spring bean definition and the `getName()` method override.
- Large file streaming: always close `ContentReader` streams in a `finally` block or use
  try-with-resources.
- Never call `nodeService.deleteNode()` or modify the record node inside the action executor —
  AGS manages record lifecycle separately after the action completes.

## How to Ask Claude for Help

- To implement a new service class: describe the inputs/outputs and which Alfresco API to use
- To debug a Spring wiring issue: paste the relevant XML bean definition and the stack trace
- To add a new metadata field to the export: reference the QName from `RecordsManagementModel`
  or `ContentModel` and ask Claude to add it to the `MetadataExport` builder
- To write an integration test: describe the AGS lifecycle step to simulate
