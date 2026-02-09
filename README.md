# gh2fizzy

Migrate GitHub Issues to Fizzy Cards - a command-line tool that transfers issues from GitHub to [Fizzy](https://fizzy.do) with full comment history preservation.

## Features

- Migrate GitHub issues to Fizzy cards with title and description
- Convert GitHub labels to Fizzy tags (auto-created)
- Preserve all comments with author attribution and timestamps
- **Multi-identity support**: Map GitHub users to Fizzy accounts to preserve authorship
- Filter issues by state, labels, assignee, or milestone
- Migrate specific issues by number
- Dry-run mode to preview migrations
- Post-migration actions: close issues and/or add labels

## Prerequisites

### Required Tools

1. **GitHub CLI (`gh`)** - [Installation](https://cli.github.com/)
   ```bash
   gh --version
   gh auth login
   ```

2. **Fizzy CLI (`fizzy`)** - [Installation](https://github.com/robzolkos/fizzy-cli/releases)
   ```bash
   fizzy --version
   fizzy auth login YOUR_TOKEN
   ```

3. **jq** - JSON processor
   ```bash
   # macOS
   brew install jq

   # Linux
   apt-get install jq  # Debian/Ubuntu
   yum install jq      # RHEL/CentOS
   ```

### Optional Tools

4. **cmark** or **pandoc** - Markdown to HTML conversion

   GitHub issues use Markdown formatting. For proper rendering in Fizzy, install one of these converters:

   ```bash
   # cmark (recommended - lightweight)
   # macOS
   brew install cmark

   # Linux
   apt-get install cmark  # Debian/Ubuntu

   # OR pandoc (full-featured)
   # macOS
   brew install pandoc

   # Linux
   apt-get install pandoc
   ```

   If neither is installed, content will be migrated as plain text (Markdown syntax visible but not rendered).

### Environment Variables (Optional)

```bash
export FIZZY_ACCOUNT="YOUR_ACCOUNT_ID"
```

## Installation

```bash
# Clone or download the script
curl -O https://raw.githubusercontent.com/robzolkos/gh2fizzy/main/gh2fizzy
chmod +x gh2fizzy

# Or move to a directory in your PATH
mv gh2fizzy /usr/local/bin/
```

## Quick Start

```bash
# Migrate all open issues to a Fizzy board
gh2fizzy --board 12345

# Preview what would be migrated (dry run)
gh2fizzy --board 12345 --dry-run

# Migrate from a specific repository
gh2fizzy --board 12345 -R owner/repo
```

## Usage

```
gh2fizzy --board BOARD_ID [OPTIONS]
```

### Required Arguments

| Argument | Description |
|----------|-------------|
| `--board BOARD_ID` | Fizzy board ID to create cards in |

### GitHub Filters

| Argument | Short | Description | Default |
|----------|-------|-------------|---------|
| `--repo` | `-R` | GitHub repository (owner/repo) | Current repo |
| `--state` | | Issue state: `open`, `closed`, `all` | `open` |
| `--label` | `-l` | Filter by label (repeatable) | None |
| `--assignee` | | Filter by assignee username | None |
| `--milestone` | | Filter by milestone title | None |
| `--limit` | | Maximum issues to migrate | 100 |
| `--issue` | `-i` | Specific issue number (repeatable) | None |

### Fizzy Options

| Argument | Description | Default |
|----------|-------------|---------|
| `--account` | Fizzy account ID | `$FIZZY_ACCOUNT` |
| `--column` | Place cards in specific column | Triage |
| `--identity-map` | JSON file mapping GitHub usernames to Fizzy auth tokens | None |
| `--default-account` | GitHub username from map to use for unmapped users and system comments | None |

### Behavior Options

| Argument | Short | Description |
|----------|-------|-------------|
| `--dry-run` | | Preview without making changes |
| `--verbose` | `-v` | Show detailed output |
| `--quiet` | `-q` | Suppress non-error output |
| `--close-issues` | | Close GitHub issues after migration |
| `--add-migrated-label` | | Add "migrated-to-fizzy" label |

## Examples

### Basic Migration

```bash
# Migrate all open issues
gh2fizzy --board 12345

# Migrate from a specific repo
gh2fizzy --board 12345 -R myorg/myrepo
```

### Filtered Migration

```bash
# Migrate only bugs
gh2fizzy --board 12345 -l bug

# Migrate issues with multiple labels
gh2fizzy --board 12345 -l bug -l urgent

# Migrate issues assigned to a user
gh2fizzy --board 12345 --assignee octocat

# Migrate closed issues
gh2fizzy --board 12345 --state closed

# Migrate all issues (open and closed)
gh2fizzy --board 12345 --state all
```

### Specific Issues

```bash
# Migrate specific issues by number
gh2fizzy --board 12345 -i 42 -i 43 -i 44
```

### With Post-Migration Actions

```bash
# Migrate and close issues in GitHub
gh2fizzy --board 12345 --close-issues

# Migrate and label issues as migrated
gh2fizzy --board 12345 --add-migrated-label

# Both: close and label
gh2fizzy --board 12345 --close-issues --add-migrated-label
```

### Targeting a Specific Column

```bash
# Place migrated cards in a specific column
gh2fizzy --board 12345 --column 67890
```

### Dry Run (Preview)

```bash
# See what would be migrated without making changes
gh2fizzy --board 12345 --dry-run --verbose
```

### Multi-Identity Support

Preserve original authorship by mapping GitHub usernames to Fizzy authentication tokens.

**Create identity-map.json:**
```json
{
  "alice-gh": "fizzy_token_alice...",
  "bob-github": "fizzy_token_bob...",
  "system-bot": "fizzy_token_bot..."
}
```

**Get tokens:** Each user generates an API token from their Fizzy profile settings.

**Run migration:**
```bash
gh2fizzy --board 12345 \
  --identity-map identity-map.json \
  --default-account system-bot
```

**Result:**
- Cards created as the GitHub issue author
- Comments created as the GitHub comment author
- Unmapped users and system events use the `system-bot` account from the map

## Data Mapping

| GitHub | Fizzy |
|--------|-------|
| Issue title | Card title |
| Issue body | Card description |
| Issue labels | Card tags (auto-created) |
| Issue comments | Individual card comments with author attribution |
| Issue author | Card creator (when using `--identity-map`) |
| Comment authors | Comment creators (when using `--identity-map`) |
| Issue references (`#123`) | Converted to Fizzy card links |
| Commit SHAs (`abc1234`) | Converted to GitHub commit links |

### Comment Format

Each GitHub comment is migrated as an individual Fizzy comment:

**Without identity mapping:**
```markdown
*Migrated from GitHub - originally by @octocat*

Original comment text here...
```

**With identity mapping:**
Comments are created directly as the mapped Fizzy user, preserving the original authorship without the attribution prefix.

Timeline events (issue closed, referenced in commits, etc.) are also migrated as comments and always use the default account.

### Issue Reference Linking

GitHub issue references (like `#123`) in issue descriptions and comments are automatically converted to Fizzy card links:

**GitHub markdown:**
```markdown
This fixes #42 and is related to #43
```

**Converted to:**
```markdown
This fixes [#42](https://fizzy.example.com/board-id/42) and is related to [#43](https://fizzy.example.com/board-id/43)
```

**Notes:**
- Only same-repository references (`#123`) are converted
- Cross-repository references (`owner/repo#123`) are preserved as-is
- References that are already part of markdown links are preserved
- Color codes like `#123ABC` are not converted (requires word boundary after number)

### Commit Reference Linking

Commit SHA references in issue descriptions and comments are automatically converted to GitHub commit links:

**GitHub markdown:**
```markdown
Fixed in d5cf4a198 and also 1234567890abcdef1234567890abcdef12345678
```

**Converted to:**
```markdown
Fixed in [`d5cf4a198`](https://github.com/owner/repo/commit/d5cf4a198) and also [`1234567890abcdef1234567890abcdef12345678`](https://github.com/owner/repo/commit/1234567890abcdef12345678)
```

**Notes:**
- Hex strings from 7-40 characters are converted (GitHub's commit SHA range)
- Must be lowercase hexadecimal (0-9, a-f)
- Must be standalone (not part of a longer hex string)
- Links point to the repository being migrated from
- Shorter hex strings (6 chars or less) and uppercase are not converted to avoid false positives

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | Missing dependencies |
| 4 | Authentication error |
| 5 | Partial failure (some issues failed) |

## Troubleshooting

### "GitHub CLI not authenticated"

```bash
gh auth login
gh auth status  # Verify
```

### "Fizzy CLI not authenticated"

```bash
# Get token from https://app.fizzy.do/my/profile
fizzy auth login YOUR_TOKEN
fizzy identity show  # Verify
```

### "Missing required tools"

Install the missing tools listed in the error message. See [Prerequisites](#prerequisites).

### "No issues found matching the criteria"

- Check if you're in a git repository (or use `-R owner/repo`)
- Verify the state filter matches your issues
- Try `gh issue list` to see available issues

### Finding Board and Column IDs

```bash
# List all boards
fizzy board list

# List columns for a board
fizzy column list --board BOARD_ID
```

## License

MIT
