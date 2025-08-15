<div align="center">

![license](https://img.shields.io/github/license/seaofvoices/rust-release-action)

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/seaofvoices)

</div>

# Rust Release Action

A GitHub Action to automate Rust crate releases. How it works:

- Automatically extracts the version from `Cargo.toml` (or use a custom version)
- Publishes crate to crates.io (optional)
- Creates a new tag and a GitHub release

**This action tightly integrates with the [Build Rust Artifact Action](https://github.com/seaofvoices/build-rust-artifact-action).** This action will give you the additional:

- Automatically launches parallel jobs to attach **pre-built** binaries to a newly created release
  - for Windows, MacOs (`x86_64` and `aarch64`) and Linux (`x86_64` and `aarch64`)

*Note*: this action is currently limited to Rust projects that aren't using workspaces. If there is a single `Cargo.toml` file at the root of your project, it is good to go.

Jump to:

- [installation](#installation)
- [inputs](#inputs)
- [outputs](#outputs)
- [full example](#example-trigger-a-full-release-manually)

## Installation

Add this action to your workflow by referencing it in your `.github/workflows/*.yml` file:

```yaml
- uses: seaofvoices/rust-release-action@v1
  with:
    publish_crate: true
  env:
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `release_tag` | Version to release (e.g., `v1.0.0` or `1.0.0`). Auto-prefixes "v" if missing. Uses Cargo.toml version if empty | No | `''` |
| `release_ref` | Branch, tag or SHA to checkout | No | `''` |
| `publish_crate` | Whether to publish the crate to crates.io | No | `true` |
| `cargo_token` | Token for publishing to crates.io (required if `publish_crate: true`) | Required when `publish_crate` is `true` | - |
| `github_token` | GitHub token for creating releases | No | `${{ github.token }}` |
| `package_name` | Package name for build artifacts (uses `package.name` from Cargo.toml if empty) | No | `''` |


## Outputs

The action produces 2 output values that can be referenced in subsequent steps:

| Output | Description |
|--------|-------------|
| `upload_url` | The upload URL for the created GitHub release, useful for uploading additional assets |
| `build_matrix` | JSON build matrix for cross-platform compilation with `os`, `name`, `artifact_name`, `cargo_target`, and optional `linker` fields |

The `build_matrix` output is meant to be used by the [Build Rust Artifact Action](https://github.com/seaofvoices/build-rust-artifact-action). See the [example](#example-trigger-a-full-release-manually) below.

## Example: Trigger a Full Release Manually

The following workflow configuration will create a button under the `Actions` tab of your repository. You will be able to click and configure the release. Triggering the workflow will:

- Create a new tag 🏷️
- Create a new release 🚢
- *Automatically* attach **pre-built binaries** to the release 🚀

Make sure to add a valid `CARGO_REGISTRY_TOKEN` value to your repository's secrets.

In a `.github/workflows/release.yml` file:

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      release_tag:
        description: 'The version to release (uses Cargo.toml if empty)'
        default: ''
        type: string

      release_ref:
        description: 'The branch, tag or SHA to checkout (default to latest)'
        default: ''
        type: string

permissions:
  contents: write

jobs:
  create-release:
    name: Publish darklua crate
    runs-on: ubuntu-latest

    outputs:
      upload_url: ${{ steps.create.outputs.upload_url }}
      build_matrix: ${{ steps.create.outputs.build_matrix }}

    steps:
      - name: Release specific version
        id: create
        uses: seaofvoices/rust-release-action@v1
        with:
          release_tag: ${{ inputs.release_tag }}
          release_ref: ${{ inputs.release_ref }}
          publish_crate: true
          cargo_token: ${{ secrets.CARGO_REGISTRY_TOKEN }}

  build-artifacts:
    needs: create-release

    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJson(needs.create-release.outputs.build_matrix) }}

    name: Build (${{ matrix.artifact-name }})
    runs-on: ${{ matrix.os }}

    steps:
      - name: Build artifact
        id: create_release
        uses: seaofvoices/rust-build-artifact-action@v1
        with:
          release_ref: ${{ inputs.release_ref }}
          upload_url: ${{ needs.create-release.outputs.upload_url }}
          package_name: ${{ matrix.name }}
          artifact_name: ${{ matrix.artifact_name }}
          cargo_target: ${{ matrix.name }}
          linker: ${{ matrix.linker }}
```

## License

This project is available under the MIT license. See [LICENSE.txt](LICENSE.txt) for details.
