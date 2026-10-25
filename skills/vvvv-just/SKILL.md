---
name: vvvv-just
description: "Write, review, or adapt Just recipes (.justfile and .just modules) for vvvv gamma / VL projects. Use for dev and build pipelines, plugin or NuGet development and debugging, vvvv.exe --open, vvvvc.exe exports, C# prerequisites, application asset staging, and optional pack pipelines."
---

# Just recipes for vvvv / VL

Design recipes around two main pipelines: **dev** and **build**. Add **pack**
only when there is a distinct packaging operation. Distinguish the two mediums
before choosing what each pipeline does:

| Medium | dev | build | Optional pack |
| --- | --- | --- | --- |
| Plugin / NuGet | Build local C# prerequisites, then open a help/example/test patch in vvvv with the package editable and debugging available. | Build prerequisites and use vvvvc to export a verification host where one exists. | Produce the distributable NuGet using the package's existing manifest/tooling. |
| Project / App | Build local C# prerequisites, then open the application patch in vvvv. | Build prerequisites, export with vvvvc, then stage required assets and runtime files into the application output. | Assemble exported applications, plugins, and assets into a distribution/archive/installer. |

A repository can contain both mediums. Keep their targets explicit rather than
forcing every target through the same packaging steps.

## Inspect first

1. Read the existing Just entry point, modules, and directly related build/package
   configuration. Reuse working commands, naming, and output conventions.
2. Identify the medium, entry patches, package IDs, local package repositories,
   C# prerequisites, DLL output locations, asset trees, and expected deliverables.
3. Inspect project references and documented dependencies. A patch can use a local
   assembly that is not reachable through another project's C# references.
4. Check installed `just --version` and tool help before introducing unfamiliar
   syntax or flags. `vvvvc.exe --help` describes the installed compiler contract.
5. Ask when a design choice remains unresolved: a plugin without a verification
   patch, a NuGet without a packing manifest, unclear asset layout, or competing
   deployment formats. Do not invent entry patches or package metadata.

Do not read or edit `.vl` patches. Obtain entry-point paths from recipes,
documentation, or the user. Never create a patch to make a recipe work.

## Directory and tool conventions

The shared directories live **one level above the repository**, not inside it:

```text
workspace-parent\
  __nugets\             shared extracted NuGet package cache
  __sources\            local NuGet package source/feed
  repository\
    .justfile
    .just\
    ...
```

- Resolve all paths from the repository root, not the caller's working directory.
  Module files live another level down; their directory is not the root.
- Use `__nugets` for `--nuget-path`. Use `__sources` as a package feed, including
  `--export-package-sources` when exported applications need that source.
- These directories have different roles. Do not place `.nupkg` files in the
  extracted cache as a substitute for a feed. Never clean either shared directory.
- Configure a vvvv installation directory once and derive both `vvvv.exe` and
  `vvvvc.exe` from it. Preserve an existing environment override such as `VVVV_DIR`.
  Do not bake this machine's installed version into a reusable template.
- Use PowerShell explicitly for Windows recipes. Prefer argument arrays and the
  call operator (`&`) for native commands, with paths supplied as single arguments.
- Paths containing spaces and semicolon-separated package lists must remain single
  arguments. Avoid splitting forwarded arguments on whitespace.
- Use the SDK architecture required by the project. Do not assume the first
  `dotnet` on PATH is suitable; an x64 host can require an explicit x64 SDK.

## Recipe organization

Keep a small project flat. For multiple entry points or reusable steps, use:

```text
.justfile               public entry point and module declarations
.just\
  common.just           shared paths/settings, when useful
  dev.just              editor launches
  build.just            compiler exports and output staging
  pack.just             only when separate packaging is needed
```

Follow the installed Just version's module/import rules. Import shared definitions
where needed; do not assume sibling modules share variable or recipe scope.

- Public targets should describe the workflow: `just dev`, `just build`, and,
  if needed, `just pack`. Multi-target projects may expose `just dev server`,
  `just build editor`, or `just pack plugin`.
- Use private helpers for C# builds, editor launches, exports, and staging.
  Keep the names distinct: compiling a DLL is not the same as exporting an app.
- Express ordering with Just prerequisites where possible. An alternative is an
  explicit helper invocation that checks failure before continuing.
- Both dev and build depend on the required local C# builds. A separate manual
  `build lib` command is useful but does not satisfy this prerequisite.
- Do not make dev depend on a full vvvvc export. Do not add services, watch loops,
  archives, installers, or extra modules unless the project needs them.

## C# prerequisites

If the repository includes C# projects used by the selected patch, build them
**before** vvvv or vvvvc loads the patch:

```text
dev target   -> required dotnet builds -> vvvv.exe --open
build target -> required dotnet builds -> vvvvc.exe -> output staging
pack target  -> required build/package inputs -> distributable
```

Choose the configuration that produces the DLLs the patch actually references.
Do not blindly switch to Release for an export: a patch may reference
`lib\Debug\<tfm>\...`. Plugin debugging also requires matching assemblies and
symbols. Preserve the established output layout.

Build the appropriate solution or explicit project set, including patch-only
dependencies. Required paths must fail clearly when missing; do not silently skip
unresolved project globs. Check every native exit code before launching/exporting.

## dev: open source in vvvv

Launch **`vvvv.exe --open <entry.vl>`**, not the exported application. Supply:

- The parent-level NuGet cache via `--nuget-path`.
- The actual local source package directories via `--package-repositories`.
- Relevant package IDs/globs via `--editable-packages`.
- Existing project-required debugging or multiple-instance flags.

### Plugin / NuGet

Open the help, example, or test patch that exercises the package. Load the local
package from source rather than accidentally testing an installed cached version.
Keep editable-package patterns narrow enough to identify the intended packages.

Build local managed code with usable symbols and preserve established `--debug`
settings. Launching vvvv does not itself attach a managed debugger; document an
existing attach workflow when required rather than inventing a debugger flag.

### Project / App

Open the selected application patch with its source dependencies available. Multiple
apps can have separate targets. Preserve necessary external service checks, but
do not infer a service dependency from an unrelated example.

For asynchronous editor launches, quote `Start-Process -ArgumentList` carefully:
it constructs a command line, so an array alone does not preserve embedded spaces.
Use the correct working directory and surface executable/launch failures.

## build: export, then make the result runnable

Use **`vvvvc.exe <entry.vl>`** after the C# prerequisites succeed. Configure:

- `--output-directory` for the target's export root.
- `--nuget-path`, `--package-repositories`, and needed editable packages.
- `--export-package-sources` with the parent `__sources` feed when applicable.
- The project's output type, runtime/architecture, icon, and asset behavior.

Preserve project-specific flags; do not universally require `WinExe`, `win-x64`,
or a particular asset mode. Fail on compiler errors rather than using
`--ignore-errors` to manufacture a successful build.

The compiler's output layout is:

```text
<output-directory>\<AppName>\<AppName>.exe
```

Do not assume the executable is directly in `--output-directory`. Resolve the
actual application directory before staging files or adding a run command.

### Plugin / NuGet

Export an existing verification/example host when this is a deliverable or build
check. This checks the plugin in an exported application; it does **not** produce
the plugin's NuGet package.

A library-only package may have no meaningful vvvvc entry point. In that case,
clarify its build contract rather than inventing an application, pretending
`dotnet build` is a vvvvc export, or treating a `.nupkg` as executable output.

### Project / App: assets are part of the build

A successful compiler exit is not the complete app build. After export:

1. Locate the actual application directory containing the executable.
2. Copy all required asset trees into their intended relative destinations,
   commonly `<application-directory>\assets`.
3. Stage required plugin outputs and approved runtime configuration alongside it.
4. Verify required files exist in the final output.

For portable new apps, consider `RelativeToOutput` and stage files consistently.
Existing projects may intentionally use `RelativeToDocument`; preserve that mode
unless changing asset resolution is explicitly agreed.

Do not flatten asset trees, nest `assets\assets`, or assume every runtime resource
is copied by vvvvc. Missing required sources must fail the build; skip only inputs
explicitly declared optional.

Do not copy a development `.env` or credentials into a distributable by default.
Use the project's approved deployment configuration. If a packaged-run helper is
needed, launch the executable with its application directory as working directory.

## Optional pack: package the right medium

Add pack only for a deliverable distinct from build:

- **Plugin / NuGet:** use the repository's actual packing mechanism, such as
  `dotnet pack` for a packable C# project or an established `.nuspec` workflow for
  a VL package. Include the required VL files, libraries, metadata, and help
  content according to the existing manifest. Do not invent a universal vvvvc
  NuGet-packing command.
- **Project / App:** depend on completed app builds and assemble their staged
  outputs, plugins, and assets. Add archives/installers only when requested.

Keep local feed publication explicit. Put packages in the parent `__sources`
when that is the configured destination; do not upload to a remote feed or modify
the shared cache implicitly. Do not use `--no-build` for packing unless the exact
configuration/framework inputs have already been built.

`--export-package-sources` only adds sources to the generated NuGet configuration:
it does not create, publish, or copy a NuGet package.

## Failure handling and verification

- Use `$ErrorActionPreference = 'Stop'` in multi-step PowerShell script recipes.
  Check `$LASTEXITCODE` immediately after native commands; the preference alone
  does not reliably turn native nonzero exit codes into terminating errors.
- Propagate failures before subsequent launches, copies, or packaging. If using
  robocopy, codes below 8 are nonfatal; codes 8 and above are failures.
- Prefer compiler-supported cleanup. Any manual cleanup must target a verified,
  target-specific generated directory, never the repository or shared folders.
- Validate Just syntax and the public command tree without running recipes, using
  the installed version's listing/show/dry-run facilities.
- Check prerequisite order for every target, not just a manually invoked helper.
- When execution is in scope, verify the actual deliverable: correct DLLs/symbols
  for dev, exported executable plus assets for build, package contents for pack.
  Do not launch the GUI or publish packages merely to validate recipe syntax.
- Read repository test instructions before running tests; do not add a full test
  suite invocation as an incidental recipe validation step.

Use these as patterns, not values to copy. Patch names, package IDs, service
requirements, SDK paths, and installed vvvv versions belong to each project.
