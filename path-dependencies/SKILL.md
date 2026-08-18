---
name: path-dependencies
description: Update all pubspec overrides to use path dependencies for every available private Dart/Flutter package found in the Developer folder. Detects available packages from ~/Developer or a configured path, and rewrites dependency_overrides to point to local paths instead of remote versions. Works with pubspec_overrides.yaml if it exists, or inline in pubspec.yaml.
---

# Path Dependencies

Use this skill when the user wants to switch dependencies to path-based overrides for local development, or to set up path dependencies for all available private packages.

## When to Use

- The user says "set up path dependencies", "use local packages", "use path overrides", "link local packages", or "switch to path"
- **Specific packages**: If the user specifies one or more package names, ONLY perform the requested switch (path or git) for those specific packages, leaving others in their current state.
- The user is working on multiple interdependent private Dart/Flutter packages and wants to develop them together
- The user wants to avoid pub.dev resolution for private packages during local development
- After cloning a project that depends on private packages
- The user asks to "comment git overrides and uncomment path overrides" or vice versa

## Configuration

The skill uses a configurable `DEVELOPER_ROOT` environment variable. If not set, defaults to `~/Developer`.

## Process

### 1. Find the project's override target file

- Check if `pubspec_overrides.yaml` exists in the project root. If it does, **use that file** for all override operations (this is where dependencies `dependency_overrides:` should go).
- If `pubspec_overrides.yaml` does NOT exist, use `pubspec.yaml` and add/update the `dependency_overrides:` section there.
- Read the current content of the target file to understand existing overrides.

### 2. Discover available packages

- List subdirectories in `$DEVELOPER_ROOT` that contain a `pubspec.yaml` file.
- For each such directory, read its `pubspec.yaml` and extract the `name:` field.
- Build a map of `package-name -> /absolute/path/to/package`.

### 3. Find dependencies that can use path overrides

- Read the project's `pubspec.yaml`.
- Look at **both** `dependencies:` and `dev_dependencies:` sections.
- Pay attention to `hosted:` URLs — packages with custom registries (e.g., `url: https://pub.zuzu.dev`) ARE candidates for path overrides just like any other package.
- For each dependency listed, check if its name exists in the map from step 2.
- A dependency matches if the package name in `pubspec.yaml` equals the name in the discovered package's `pubspec.yaml`.

### 4.1 Granular Switching (New)

When the user specifies only certain packages:

- Locate the entry for the specified package in BOTH the `PATH OVERRIDES` and `GIT OVERRIDES` sections.
- If switching **TO path**: uncomment the entry in `PATH OVERRIDES` and comment out the entry in `GIT OVERRIDES`.
- If switching **TO git**: comment out the entry in `PATH OVERRIDES` and uncomment the entry in `GIT OVERRIDES`.
- Do NOT touch other packages in the override file.

1. **Always-active section** — version pins and fork-only git overrides (no path alternative). These are never touched.
2. **PATH OVERRIDES section** — entirely commented out by default. Each entry uses `path:`.
3. **GIT OVERRIDES section** — active by default. Each entry uses `git:`.

Both swappable sections are clearly labeled with header comments. Swapping is a single operation:

- **Switch TO path**: comment out the entire GIT OVERRIDES section, uncomment the entire PATH OVERRIDES section.
- **Switch TO git**: comment out the entire PATH OVERRIDES section, uncomment the entire GIT OVERRIDES section.

```yaml
dependency_overrides:
  # Always-active (never touch these):
  analyzer: ^13.3.0

  # ==========================================================
  # PATH OVERRIDES
  #   Uncomment these + comment GIT OVERRIDES to use paths
  # ==========================================================
  # shadcn_ui:
  #   path: /Users/.../shadcn_ui

  # ==========================================================
  # GIT OVERRIDES
  #   Uncomment these + comment PATH OVERRIDES to use git
  # ==========================================================
  shadcn_ui:
    git:
      url: https://github.com/example/shadcn_ui.git
      ref: main
```

### 5. Add new packages to both sections

When a new local package is discovered that has both a path and a git override:

- Add the `path:` entry to the PATH OVERRIDES section (comment it out if currently in git mode).
- Add the `git:` entry to the GIT OVERRIDES section (uncommented if currently in git mode).
- If the package has no known git URL, check `cd ~/Developer/<package> && git remote -v`.
- If there's no git remote either, add only to the PATH OVERRIDES section (the package will resolve from its hosted version when path overrides are inactive).

### 6. Preserve existing non-swappable overrides

Always preserve:

- Version pins (e.g., `analyzer: ^13.3.0`)
- Fork-only git overrides that have no path alternative (e.g., `intl`, `puppeteer`)

### 7. Run pub get

After writing the overrides, run `flutter pub get` (or `dart pub get` for non-Flutter projects) to verify the overrides resolve correctly. If the project uses private registries, include the auth token:

```bash
PUB_ZUZU_TOKEN=$PUB_ZUZU_TOKEN flutter pub get
```

### 8. Report

Report back:

- Which target file was modified (`pubspec_overrides.yaml` or `pubspec.yaml`)
- Which packages were switched from git to path (or vice versa)
- Any new packages added to the sections
- Whether `pub get` succeeded or failed (with error details if it failed)
- Any packages that were listed as dependencies but NOT found locally (these remain as remote/hosted versions)
