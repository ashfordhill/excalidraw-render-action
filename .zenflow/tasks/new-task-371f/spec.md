# Technical Specification: Replace Excalidraw Export Tool

## Difficulty Assessment
**Medium** - The task requires replacing a third-party dependency with an alternative solution. While the implementation is straightforward (modifying the GitHub Action's bash script), it requires:
- Evaluating multiple alternative tools
- Understanding Playwright installation in CI/CD environments
- Ensuring the new tool produces correct output
- Maintaining backward compatibility with existing inputs/outputs

## Technical Context

### Current Implementation
- **Language**: Bash (composite GitHub Action)
- **Runtime**: Node.js 20 in GitHub Actions
- **Environment**: Ubuntu latest runners
- **Current dependency**: `@tommywalkie/excalidraw-cli` (npm package)
- **File structure**:
  - `action.yml` - GitHub Action definition (composite action)
  - `test.excalidraw` - Sample test file
  - `.github/workflows/commit-rendered-files.yml` - Test workflow

### Problem Statement
The current tool (`@tommywalkie/excalidraw-cli`) produces PNG files with:
- Missing styling from the original .excalidraw files
- Incorrect arrow placements
- Other rendering inconsistencies

When using the export option on excalidraw.com, the PNG renders correctly with all styling and proper element placement.

### Available Alternative Solutions

After researching, three main alternatives exist:

1. **excalidraw-brute-export-cli** (npm: `excalidraw-brute-export-cli`)
   - Uses Playwright + Firefox to run headless browser
   - Exports using the exact same process as excalidraw.com
   - **Pros**: Most reliable rendering, matches excalidraw.com exactly, addresses known bugs in other tools
   - **Cons**: Slower (browser overhead), requires Playwright installation, larger dependency footprint
   - **Status**: Actively maintained, published to npm and Docker
   - **CI/CD Ready**: Yes, includes Docker images

2. **excalidraw-to-svg** (npm: `excalidraw-to-svg`)
   - Uses jsdom to simulate DOM, runs Excalidraw+React in Node.js
   - **Pros**: Faster, lighter weight
   - **Cons**: Known bugs with edge-labels and arrow placement (the exact issues we're experiencing)
   - **Status**: Last published 2 years ago

3. **excalidraw_export** (GitHub: Timmmm/excalidraw_export)
   - Similar to excalidraw-to-svg but with font embedding
   - **Pros**: Better font handling
   - **Cons**: Still has known arrow/edge rendering issues, less actively maintained
   - **Status**: Available but not on npm registry

## Recommended Solution

**Use `excalidraw-brute-export-cli`** for the following reasons:

1. **Correctness**: It uses the exact same export process as excalidraw.com, which is confirmed to work correctly
2. **Addresses the specific issue**: Specifically designed to avoid rendering bugs that affect other tools
3. **CI/CD Ready**: Already tested and working in GitHub Actions environments
4. **Maintained**: Active development and Docker images available
5. **Not a "last resort"**: While it uses Playwright (which the user mentioned as last resort), it's a purpose-built solution that directly addresses the rendering issues

The performance overhead of Playwright is acceptable for a GitHub Action that runs on file changes (not frequently), and correctness is more important than speed in this use case.

## Implementation Approach

### Changes to `action.yml`

The composite action needs three modifications:

1. **Replace npm installation step**:
   - Remove: `npm install -g @tommywalkie/excalidraw-cli`
   - Add: `npm install -g excalidraw-brute-export-cli`

2. **Add Playwright installation step**:
   ```yaml
   - name: Install Playwright dependencies
     shell: bash
     run: |
       npx playwright install-deps firefox
       npx playwright install firefox
   ```

3. **Update conversion command**:
   - Replace `excalidraw-cli "$relative_file" "$output_dir"` with appropriate `excalidraw-brute-export-cli` command
   - The new command format:
     ```bash
     excalidraw-brute-export-cli \
       -i "$relative_file" \
       -o "$output_file" \
       --format png \
       --background 1 \
       --embed-scene 0 \
       --dark-mode 0 \
       --scale 1
     ```

### Output Format Differences

- **Current tool**: Outputs to directory, derives filename from input
- **New tool**: Requires explicit output file path
- **Compatibility consideration**: Need to compute full output path instead of just output directory

### Configuration Options to Expose

The new tool supports additional options that could be exposed as inputs (future enhancement):
- `--background` (include background, default: true)
- `--embed-scene` (embed scene data in PNG, default: false)
- `--dark-mode` (export in dark mode, default: false)
- `--scale` (export scale, default: 1)

For initial implementation, use sensible defaults. Could add as optional inputs later if needed.

## Source Code Structure Changes

### Files to Modify
1. `action.yml` - Primary implementation file
   - Lines 25-27: Update npm installation
   - Lines 29-78: Update conversion script

### Files to Keep Unchanged
1. `.github/workflows/commit-rendered-files.yml` - No changes needed (uses the action's inputs)
2. `test.excalidraw` - Keep for testing
3. `README.md` - Should be updated to reflect new dependency, but implementation only

### No New Files Required
All changes are modifications to existing `action.yml`.

## Data Model / API / Interface Changes

### Action Inputs (Unchanged)
- `input-dir`: Directory containing .excalidraw files (default: '.')
- `output-dir`: Directory to output PNG files (default: same as input)

### Action Outputs
- None defined currently, none needed

### Breaking Changes
- None - the interface remains the same
- PNG output filenames remain the same
- Directory structure preservation remains the same

## Verification Approach

### Manual Testing
1. Test with `test.excalidraw` file in the repository
2. Compare output PNG with:
   - Current tommywalkie tool output
   - excalidraw.com export output
3. Verify styling and arrow placement are correct

### Automated Testing
The repository has an existing workflow (`.github/workflows/commit-rendered-files.yml`) that:
- Triggers on push to main when .excalidraw files change
- Runs the action
- Commits the generated PNG

**Test procedure**:
1. Create a test branch
2. Modify `test.excalidraw` or add a new test file
3. Push changes and trigger the workflow
4. Verify the generated PNG is committed correctly
5. Visual inspection of the generated PNG

### Lint/Typecheck Commands
- No specific lint/typecheck needed (bash script)
- Validation: YAML syntax validation for `action.yml`
- Can use: `yamllint action.yml` if available

### Success Criteria
1. The action successfully converts .excalidraw files to PNG
2. Generated PNGs have correct styling (colors, fills, strokes)
3. Arrow placements are accurate
4. The GitHub Action workflow completes successfully
5. No regression in directory structure handling

## Potential Risks and Mitigations

### Risk: Playwright Installation Time
- **Impact**: Slower action execution
- **Mitigation**: This is acceptable for a file-change-triggered workflow. If performance becomes an issue, can cache Playwright installation.

### Risk: Playwright Dependency Size
- **Impact**: Larger disk usage in GitHub Actions
- **Mitigation**: GitHub Actions runners have sufficient space. Alternative: use the Docker image instead.

### Risk: Firewall/Network Issues
- **Impact**: Playwright might need to download browser binaries
- **Mitigation**: The `playwright install` command handles this. GitHub Actions runners have network access.

### Risk: Breaking Changes in excalidraw-brute-export-cli
- **Impact**: Future updates might break compatibility
- **Mitigation**: Pin to specific version initially, test updates before upgrading

## Alternative Implementation: Docker

If Playwright installation causes issues, can use the Docker approach:

```yaml
- name: Convert Excalidraw files to PNG
  shell: bash
  run: |
    docker run --rm \
      -v "${{ github.workspace }}:/data" \
      ghcr.io/realazthat/excalidraw-brute-export-cli:v0.4.0 \
      -i "/data/$relative_file" \
      -o "/data/$output_file" \
      --format png
```

This avoids Playwright installation but requires Docker (which is available on GitHub Actions runners).

## References

- [excalidraw-brute-export-cli GitHub](https://github.com/realazthat/excalidraw-brute-export-cli)
- [excalidraw-brute-export-cli npm](https://www.npmjs.com/package/excalidraw-brute-export-cli)
- [Excalidraw Export Utilities](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/utils/export)
- [Playwright Documentation](https://playwright.dev/)
