I'll create a comparison table of these cleaning commands to help identify the most interesting and useful ones.

| Command | Interest Level | Primary Use Case | Automation Support | Unique Value |
|---------|---------------|------------------|-------------------|--------------|
| **references** | ⭐⭐⭐⭐⭐ | Master orchestrator for multiple cleaning operations | ✅ Yes | Combines multiple cleaners, interactive mode, configurable |
| **minimizeprofiles** | ⭐⭐⭐⭐⭐ | Remove profile attrs that belong on Permission Sets | ✅ Yes | Critical for Spring 26 compliance, best practice enforcement |
| **xml** | ⭐⭐⭐⭐⭐ | Remove XML elements using XPath patterns | ✅ Yes | Highly flexible, solves repetitive XML cleaning issues |
| **flowpositions** | ⭐⭐⭐⭐ | Normalize Flow coordinates to reduce conflicts | ✅ Yes | Dramatically reduces merge conflicts in auto-layout flows |
| **destructivechanges** | ⭐⭐⭐⭐ | Remove files listed in destructiveChanges.xml | ✅ Yes (via references) | Keeps repo clean, prevents deployment errors |
| **sensitive-metadatas** | ⭐⭐⭐⭐ | Hide certificates and credentials | ✅ Yes | Security-critical, prevents credential leaks |
| **manageditems** | ⭐⭐⭐⭐ | Remove managed package metadata by namespace | ❌ Manual | Essential for orgs with managed packages |
| **orgmissingitems** | ⭐⭐⭐⭐ | Clean based on what exists in target org | ❌ Manual | Maintains org/repo consistency, reduces errors |
| **standarditems** | ⭐⭐⭐ | Remove standard objects/fields | ❌ Manual | Reduces repo size, one-time cleanup |
| **systemdebug** | ⭐⭐⭐ | Remove/comment System.debug() statements | ❌ Manual | Code hygiene, performance optimization |
| **listviews** | ⭐⭐⭐ | Change ListView filters from Mine to Everything | ❌ Manual | Fixes deployment validation issues |
| **emptyitems** | ⭐⭐⭐ | Remove empty metadata files | ❌ Manual | Cleanup after retrieval operations |
| **hiddenitems** | ⭐⭐ | Remove (hidden) temporary files | ❌ Manual | Edge case cleanup |
| **filter-xml-content** | ⭐⭐⭐ | Advanced XML filtering via JSON config | ❌ Manual | Complex deployment scenarios |
| **retrievefolders** | ⭐⭐ | Retrieve folder-based metadata | ❌ Manual | Utility for selective retrieval |

## Most Interesting Commands (Top 5):

### 🥇 **1. references** - The Swiss Army Knife
- **Why**: Orchestrates all other cleaners, provides interactive selection, persistent config
- **Impact**: One command to rule them all
- **Best For**: Setting up automated cleaning workflows

### 🥈 **2. minimizeprofiles** - Future-Proof Your Profiles
- **Why**: Addresses Salesforce's Spring 26 deprecation, enforces best practices
- **Impact**: Prevents future breaking changes, improves security model
- **Best For**: Organizations preparing for Profile deprecation

### 🥉 **3. xml** - Ultimate XML Surgeon
- **Why**: XPath + Glob patterns = unlimited cleaning possibilities
- **Impact**: Solves repetitive manual XML editing permanently
- **Best For**: Complex, recurring XML cleanup needs

### 🏅 **4. flowpositions** - Merge Conflict Eliminator
- **Why**: Transforms flows to eliminate positional conflicts
- **Impact**: Saves hours in merge conflict resolution
- **Best For**: Teams with multiple developers working on flows

### 🏅 **5. sensitive-metadatas** - Security Guardian
- **Why**: Prevents credentials from entering version control
- **Impact**: Critical security protection
- **Best For**: Every project (security requirement)

## Quick Decision Guide:

**Use immediately if you have:**
- ✅ Merge conflicts in Flows → `flowpositions`
- ✅ Profiles with permissions → `minimizeprofiles`
- ✅ Certificates in repo → `sensitive-metadatas`
- ✅ Managed packages → `manageditems`
- ✅ Same XML issues repeatedly → `xml`

**Set up automation for:**
- `references` (with `autoCleanTypes` in `.sfdx-hardis.yml`)
- `flowpositions`, `minimizeprofiles`, `sensitive-metadatas`

**Use situationally:**
- `orgmissingitems` - Before major deployments
- `systemdebug` - Before production releases
- `standarditems` - After org retrievals
- `listviews` - When deployment fails on ListView validation

Let me provide a clear explanation of each command in the clean folder:

## Clean Commands in clean

The clean commands folder contains **14 specialized commands** that help maintain a clean, efficient, and conflict-free Salesforce DX project. Here's what each command does:

### 1. **emptyitems** - Clean Empty Metadata
Removes metadata files that contain no meaningful data:
- Empty Global Value Set Translations
- Empty Standard Value Sets  
- Empty Sharing Rules
- Checks XML content for specific tags to determine if files are truly empty

### 2. **filter-xml-content** - Advanced XML Filtering
Filters XML metadata files based on JSON configuration rules:
- Removes specific XML elements based on configurable patterns
- Useful for deploying subsets of metadata
- Processes files from input folder to output folder
- Supports custom filtering rules for complex deployments

### 3. **flowpositions** - Normalize Flow Positions
Replaces all position coordinates in Auto-Layout Flows with zeros:
- Changes `<locationX>380</locationX>` to `<locationX>0</locationX>`
- Simplifies merge conflict management
- Only affects flows with `AUTO_LAYOUT_CANVAS` setting
- Can be automated via `.sfdx-hardis.yml` config

### 4. **hiddenitems** - Remove Hidden Files
Deletes files marked as hidden or temporary:
- Targets files starting with `(hidden)` content
- Removes entire LWC/Aura component folders if hidden
- Scans `.app`, `.cmp`, `.evt`, `.tokens`, `.html`, `.css`, `.js`, `.xml` files

### 5. **listviews** - Fix ListView Filters
Replaces "Mine" with "Everything" in ListView filters:
- Helps deployments pass validation
- Logs replacements in `.sfdx-hardis.yml`
- Maintains list of modified views for restoration

### 6. **manageditems** - Remove Managed Package Items
Removes metadata from specific managed package namespaces:
- Requires `--namespace` flag (e.g., `crta`)
- Preserves local customizations within managed folders
- Intelligently keeps folders containing custom items
- Useful for cleaning up org retrievals with managed packages

### 7. **minimizeprofiles** - Clean Profile Permissions
Removes profile attributes that should be on Permission Sets:
- Removes: `classAccesses`, `fieldPermissions`, `objectPermissions`, `pageAccesses`, etc.
- Follows Salesforce's Spring 26 deprecation guidance
- Configurable via `.sfdx-hardis.yml` (`skipMinimizeProfiles`)
- Keeps `userPermissions` on Admin profiles

### 8. **orgmissingitems** - Clean Based on Org State
Removes metadata not present in target org:
- Compares local project with target org's metadata
- Cleans `reportType-meta.xml` files specifically
- Removes references to deleted fields/objects
- Helps maintain consistency with org state

### 9. **references** - Master Cleaning Command
Orchestrates multiple cleaning operations:
- Supports cleaning types: `dashboards`, `destructivechanges`, `flowPositions`, `minimizeProfiles`, `sensitiveMetadatas`, etc.
- Interactive selection of cleaning operations
- Can use JSON config or destructiveChanges.xml
- Saves preferences to `.sfdx-hardis.yml` for automation

### 10. **retrievefolders** - Retrieve Folder Metadata
Retrieves specific folders from Salesforce org:
- Targets: Dashboards, Documents, Email Templates, Reports
- Preserves folder structure
- Uses selective retrieval for efficiency
- Executes `sf project retrieve start` for each folder

### 11. **sensitive-metadatas** - Hide Sensitive Data
Replaces sensitive content with placeholders:
- Detects certificates (`.crt` files)
- Replaces certificate content with "HIDDEN_BY_SFDX_HARDIS"
- Prevents credentials from being stored in Git
- Can be automated via `autoCleanTypes` config

### 12. **standarditems** - Remove Standard Objects/Fields
Cleans unwanted standard Salesforce metadata:
- Removes standard object folders without custom fields
- Keeps folders with custom fields, removes only standard fields
- Reduces repository size
- Avoids conflicts with standard Salesforce metadata

### 13. **systemdebug** - Clean Debug Statements
Removes or comments out `System.debug()` calls:
- Default: Comments out debug lines with `//`
- `--delete` flag: Completely removes debug lines
- Scans all `.cls` and `.trigger` files
- Respects `NOPMD` markers to keep intentional debug statements

### 14. **xml** - Remove XML Elements via XPath
Advanced XML cleaning using Glob patterns and XPath:
- Uses glob patterns to find XML files (e.g., `/**/*.flexipage-meta.xml`)
- Uses XPath to identify elements to remove
- Can be configured in `.sfdx-hardis.yml` via `cleanXmlPatterns`
- Powerful for repetitive XML cleaning tasks

### Common Features Across Commands:
- **Automation Support**: Many can be automated via `.sfdx-hardis.yml` `autoCleanTypes`
- **Dry Run Safe**: Most log what they're doing before making changes
- **Configurable**: Support flags like `--folder`, `--debug`
- **Project-Aware**: Require `requiresProject = true`
- **Git-Friendly**: Designed to reduce merge conflicts

These commands are essential for maintaining clean Salesforce DX projects in CI/CD pipelines and reducing deployment conflicts.
