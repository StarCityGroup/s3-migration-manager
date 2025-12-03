# S3 Migration Manager

Terminal UI for browsing S3 buckets, defining object masks, and managing storage tier transitions or restores without leaving the shell.

## Features

- **Bucket & object browser**: list all accessible buckets and their objects, including size and current storage class.
- **Lazy loading**: intelligently loads objects in batches of 200 for fast performance with large buckets (10k+ objects).
- **Auto-loading**: bucket selection triggers object loading after 1 second - no manual Enter key needed.
- **Mask-driven selection**: build prefix/suffix/contains/regex masks, test matches live, and reuse them when defining migration policies.
- **Smart pagination**: automatically loads more objects when scrolling near the end or when masks need more matches.
- **Storage class transitions**: interactively choose a target tier; confirmations include an option to request restores before the copy operation.
- **Restore workflow**: request temporary Glacier restores (default 7 days) for the current selection.
- **Policy library**: persist mask + target-class rules to `~/.config/s3-migration-manager/policies.json` for later reuse or auditing.
- **Deep storage visibility**: refresh metadata for any object to fetch its latest restore status before acting.
- **Structured object list**: fixed-width columns with restore status indicators (✓ available, ⟳ in progress, ✗ expired).

## Requirements

- Rust 1.78+ (toolchain with `cargo`).
- AWS credentials/profile accessible via the standard SDK lookup chain (env vars, `~/.aws/credentials`, SSO, etc.).

## Getting Started

```bash
cargo run
```

The first launch will download crates, create a config directory if needed, and enter the TUI.

## How It Works - Workflow Guide

### Basic Workflow

1. **Browse Buckets**: The app loads all accessible S3 buckets on startup
2. **Select Bucket**: Use arrow keys to select a bucket - objects auto-load after 1 second
3. **Filter with Masks**: Create patterns to filter objects (e.g., "logs-2024-*")
4. **Take Actions**: Transition storage classes, request restores, or save policies
5. **Reuse Policies**: Apply saved masks+transitions without recreating them

**Performance Note**: For large buckets (10k+ objects), the app loads objects in batches of 200. It shows "X of Y objects" and automatically fetches more as you scroll or when masks need additional matches.

### Navigation

- **`Tab` / `Shift+Tab`**: Switch between panes (Buckets → Objects → Policies)
- **Arrow keys**: Move selection up/down (objects auto-load when you select a bucket)
- **`[` / `]`**: Cycle through regions
- **`PgUp` / `PgDn`**: Jump 5 items at a time
- **`Home` / `End`**: Jump to first/last item
- **`Enter`**: Apply selected policy (in Policies pane)

### UI Layout

The interface is organized for efficient workflow:

```
┌─────────────────────────────────────────────────────┬──────────────┐
│ Bucket/Region Selector (compact)                    │              │
├─────────────────────────────────────────────────────┤   Policies   │
│ Filter Mask (active mask info)                      │   (saved)    │
├─────────────────────────────────────────────────────┤              │
│                                                      │              │
│ Objects List (200 of 15,342) ⟳                      │              │
│ ► file001.txt       1.23 KB  STANDARD               │              │
│   file002.txt       4.56 KB  GLACIER_IR        [✓]  │              │
│   ...                                                │              │
├─────────────────────────────────────────────────────┤              │
│ Selected Object Details                             │              │
└─────────────────────────────────────────────────────┴──────────────┘
│ Status Messages                                                    │
├────────────────────────────────────────────────────────────────────┤
│ Command Bar (keyboard shortcuts)                                  │
└────────────────────────────────────────────────────────────────────┘
```

- **Top**: Compact bucket selector with region filter
- **Middle-Left**: Filter mask status, objects list, and selected object details
- **Right**: Saved policies for quick reuse
- **Bottom**: Status log and command hints

**Object List Format**:
- Fixed-width columns for consistent alignment
- Sizes always shown in KB (e.g., "1,234.56 KB")
- Restore indicators: `[✓]` available, `[⟳]` in progress, `[✗]` expired

### Object Filtering with Masks

Masks let you select multiple objects matching a pattern:

1. **Create a mask**: Press `m` to open the mask editor
2. **Configure the filter**:
   - **Name**: Type to replace "Untitled mask" (placeholder auto-clears on first keystroke)
   - **Pattern**: The text to match (e.g., "logs-2024-")
   - **Mode**: Use `←/→` or `Space` to cycle through: Prefix, Suffix, Contains, or Regex
   - **Case**: Use `←/→` or `Space` to toggle case-sensitive matching on/off
3. **Navigate fields**: Press `Tab` to move forward, `Shift+Tab` to move backward
4. **Apply**: Press `Enter` to apply the mask, `Esc` to cancel
5. **Clear active mask**: Press `Esc` (while browsing) to remove the filter

**Mask Editor Tips**:
- Type normally in Name and Pattern fields - all characters work (no special hotkeys)
- Use arrow keys or space to change Mode and Case settings
- The Name field placeholder clears automatically when you start typing

**Important**: When a mask is active, all operations (transitions, restores) apply to **all matching objects**, not just the selected one.

### Storage Operations

#### Transitioning Storage Classes

1. Select object(s) - either:
   - Single object: Just highlight it in the Objects pane
   - Multiple objects: Apply a mask first
2. Press `s` to start storage class selection
3. Choose target class (Standard, Standard-IA, Glacier, etc.)
4. Confirm the operation
5. **Optional**: Press `o` during confirmation to request restore before transition (for archived objects)

#### Requesting Restores

For objects in Glacier/Deep Archive storage:

1. Select object(s) (single or via mask)
2. Press `r` to request a 7-day restore
3. Confirm the operation
4. Press `i` (inspect) later to check restore status

### Working with Policies

Policies save your mask + target storage class for reuse:

#### Saving a Policy

1. Create and apply a mask (`m`)
2. Select the bucket you want to use
3. Press `p` to save as policy
4. Choose the target storage class
5. Confirm - policy is saved to `~/.config/s3-migration-manager/policies.json`

#### Using Saved Policies

Navigate to the Policies pane (press `Tab` until focused), then:

- **Apply as-is**:
  1. Select policy with arrow keys
  2. Press `Enter` to apply the mask and start transition
  3. Confirm the operation

- **Edit before using**:
  1. Select policy with arrow keys
  2. Press `e` to load the mask into the editor
  3. Modify the pattern/settings as needed
  4. Press `Enter` to apply the modified mask

**Note**: Policies remember the bucket name - make sure you have the correct bucket selected before applying.

### Other Commands

| Key | Action |
| --- | --- |
| `i` | Inspect selected object (refresh metadata via HeadObject) |
| `f` | Refresh the bucket list |
| `l` | Toggle status log (view full error messages and history) |
| `?` | Toggle help screen |
| `q` / `Ctrl+C` | Quit application |
| `Esc` | Clear active mask, or close dialogs/popups |

## Storage Policies

Saved policies live at `~/.config/s3-migration-manager/policies.json`. Each entry records:

- Bucket name
- Mask definition
- Desired destination storage class
- Whether a restore should run before transition
- Timestamp and optional notes

You can version-control this file or edit it manually if needed.

## Testing & Validation

- `cargo check` (run during development) ensures the project builds and dependencies resolve.
- Most behavior depends on live AWS APIs; prefer running against a test account or buckets with dummy data before touching production buckets.

## Performance

The app is optimized for large S3 buckets:

- **Fast counting**: Quickly determines total object count without loading full details
- **Lazy loading**: Loads objects in batches of 200 for responsive UI
- **Smart prefetching**: Automatically loads more when:
  - Scrolling near the end of the list
  - Active mask has fewer than 100 matches and more objects are available
- **Memory efficient**: Only keeps loaded objects in memory, not the entire bucket
- **Non-blocking**: Background loading doesn't freeze the UI

Tested with buckets containing 100,000+ objects.

## Next Steps

Ideas for follow-up iterations:

1. Tag-based and size/date filters alongside the current key-based masks.
2. Background task queue so long copy/restore operations don't block the UI.
3. Mask-aware byte size estimations before executing transitions.
4. Optional cost estimation per plan using cached pricing tables.
5. CloudTrail-friendly dry-run mode that just logs intended actions.
6. Bulk operations with progress tracking and retry logic.
