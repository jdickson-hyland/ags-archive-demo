# Alfresco AGS Filesystem Archive Extension

This project implements a custom **Alfresco Governance Services (AGS) disposition action** that archives records to the local filesystem when an accession step fires in an AGS disposition schedule. It is built on the Alfresco In-Process SDK 4.4 targeting Alfresco Enterprise 25.x.

## What It Does

When an accession disposition step is triggered on a record or record folder, the extension:

1. Resolves a directory path mirroring the AGS File Plan hierarchy: `{archive.root}/{category}/{folder}/{record-identifier}/`
2. Streams the record's binary content to `content.{ext}`
3. Exports all node properties as `metadata.json`
4. Writes retention schedule details as `disposition.json`
5. Captures an AGS audit trail snapshot as `audit.json`
6. Stamps the record node with `archive:status = ARCHIVED` and `archive:path` (post-commit, in a new transaction)

All filesystem writes happen in `afterCommit()` — never inside the accession transaction itself — so a rollback cannot produce partial archive output.

## Project Structure

```text
ags-archive-demo-platform/
  src/main/java/
    org/hyland/archive/
      action/   ArchiveDispositionActionExecuter.java   # AGS disposition action
      service/  ArchiveService.java                     # Filesystem write logic
      model/    ArchiveAspect.java                      # archive: namespace QName constants
  src/main/resources/
    alfresco/module/ags-archive-demo-platform/
      model/    archive-model.xml                       # archive:archiveStatus aspect definition
      context/  service-context.xml                     # Spring bean wiring
      messages/ content-model.properties                # Aspect/property labels
      alfresco-global.properties                        # Module property defaults
ags-archive-demo-platform-docker/
  src/main/docker/
    alfresco-global.properties                          # Docker runtime property overrides
```

## Configuration

The following properties can be overridden in `alfresco-global.properties`:

| Property | Default | Description |
| --- | --- | --- |
| `archive.filesystem.root` | `/opt/alfresco/archive` | Root directory for all archived records |
| `archive.include.audit` | `true` | Write an AGS audit snapshot alongside content and metadata |
| `archive.failOnError` | `true` | Fail the accession step if the filesystem write fails |

## Running the Project

All services run as Docker containers. Use the `run.sh` (or `run.bat`) script:

| Command | Description |
| --- | --- |
| `./run.sh build_start` | Build the project, rebuild Docker images, start the environment and tail logs |
| `./run.sh build_start_it_supported` | As above, but also includes integration test dependencies |
| `./run.sh start` | Start the environment without rebuilding |
| `./run.sh stop` | Stop the environment |
| `./run.sh purge` | Stop containers and delete all persistent data (volumes) |
| `./run.sh tail` | Tail logs from all running containers |
| `./run.sh reload_acs` | Rebuild and restart the ACS container only (faster iteration) |
| `./run.sh reload_share` | Rebuild and restart the Share container only |
| `./run.sh build_test` | Build, start, run integration tests, then stop |
| `./run.sh test` | Run integration tests against an already-running environment |

## Build Commands

```bash
# Build without running tests
mvn clean package -DskipTests

# Build and start with Docker
mvn clean install -Prun

# Run unit tests only
mvn test

# Run integration tests against a running instance
mvn integration-test
```

## Testing the Archive Disposition Action

### Unit Tests

Unit tests for `ArchiveService` are in `ArchiveServiceTest.java` and use Mockito to mock all Alfresco services. They run without a running Alfresco instance:

```bash
mvn test -pl ags-archive-demo-platform
```

The tests cover:

- Correct archive directory path resolution from the AGS hierarchy
- Binary content written to `content.{ext}`
- `metadata.json` written with all node properties
- `disposition.json` written
- `audit.json` written (and skipped when `archive.include.audit=false`)
- Idempotency: a second call on the same record is a no-op
- Node stamped with `archive:status` and `archive:path` after write

### Manual End-to-End Test

Prerequisites: a running Alfresco Enterprise 25.x instance with AGS. Start with `./run.sh build_start` and wait for the stack to be fully up.

1. Open Share at `http://localhost:8180/share` and log in as `admin`.
1. Navigate to **RM Site** (create one if it does not exist).
1. In the **File Plan**, create a **Record Category** (e.g. `Test Category`).
1. Inside it, create a **Record Folder** (e.g. `Test Folder`).
1. Upload a file to the folder and **Declare** it as a record.
1. On the Record Category, open **Edit Disposition Schedule** and add a disposition step:
   - Action: **Accession**
   - Set the step to use action name `archive`
1. Process the disposition step (either via the scheduled job or by triggering it manually from the record folder's actions menu).
1. Verify the output on the ACS container filesystem:

```bash
docker exec -it ags-archive-demo-acs \
  find /opt/alfresco/archive -type f | sort
```

Expected output:

```text
/opt/alfresco/archive/Test_Category/Test_Folder/{record-identifier}/audit.json
/opt/alfresco/archive/Test_Category/Test_Folder/{record-identifier}/content.pdf
/opt/alfresco/archive/Test_Category/Test_Folder/{record-identifier}/disposition.json
/opt/alfresco/archive/Test_Category/Test_Folder/{record-identifier}/metadata.json
```

1. Verify the record node has been stamped. In Share, open the record's properties and confirm the **Archive Status** aspect is present with `status = ARCHIVED` and a populated `path`.

### Verifying Idempotency

Trigger the accession step a second time on the same record. Confirm:

- No duplicate or corrupted files appear in the archive directory
- The node properties are unchanged

## Enterprise Repository Access

Enterprise artifacts require credentials in `~/.m2/settings.xml`:

```xml
<server>
  <id>alfresco-private-repository</id>
  <username>YOUR_ENTERPRISE_USERNAME</username>
  <password>YOUR_ENTERPRISE_PASSWORD</password>
</server>
```

The private repository URL is `https://artifacts.alfresco.com/nexus/content/groups/private/`.

## Notes

- Spring bean wiring uses explicit XML — no `@Component` / `@Service` annotations (Alfresco convention)
- The `archive` bean ID is load-bearing: it sets the AGS action name via `BeanNameAware`
- The `alfresco-governance-services-enterprise-repo` dependency is `provided` scope — supplied by the running ACS instance
- Never call `nodeService.deleteNode()` inside the action executor; AGS manages the record lifecycle after the action completes
