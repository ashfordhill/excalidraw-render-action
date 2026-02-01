# Excalidraw Render Action

A GitHub Action that automatically converts `.excalidraw` files to PNG images using [excalidraw-cli](https://github.com/tommywalkie/excalidraw-cli).

## How It Works

1. The action searches for all `.excalidraw` files in the specified input directory (recursively)
2. For each file found, it converts it to PNG using excalidraw-cli
3. PNG files are saved with the same name as the source `.excalidraw` file
4. Directory structure is preserved when using subdirectories

## Usage

Convert all `.excalidraw` files in your repository to PNG images in the same directory:

```yaml
name: Render and Commit Excalidraw Files

on:
  push:
    branches: [ main ]
    paths:
      - '**.excalidraw'

jobs:
  render-and-commit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Render Excalidraw files to PNG
        uses: ashfordhill/excalidraw-render-action@v1
        with:
          # Make sure repo's settings in Actions -> General -> Workflow permissions has "Read and write permissions" enabled
          token: ${{ secrets.GITHUB_TOKEN }}
          input-dir: '.'
      
      - name: Commit rendered images
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git add '*.png'
          if ! git diff --cached --quiet; then
            git commit -m 'chore: update rendered excalidraw images'
            git push
          fi
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `input-dir` | Directory containing `.excalidraw` files | No | `.` |
| `output-dir` | Directory to output PNG files (defaults to same as input directory) | No | `` |

## Example

If you have the following structure:
```
.
├── diagram1.excalidraw
└── docs/
    └── diagram2.excalidraw
```

Running the action with default settings will produce:
```
.
├── diagram1.excalidraw
├── diagram1.png
└── docs/
    ├── diagram2.excalidraw
    └── diagram2.png
```
See this repo's past commits for examples of this in action with `test.excalidraw`.

## Credits

This action uses [excalidraw-cli](https://github.com/tommywalkie/excalidraw-cli) for rendering Excalidraw files.