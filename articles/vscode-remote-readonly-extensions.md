<!---
title: "Why VS Code can't install extensions to a remote when your local ones are read-only"
seoTitle: "Why VS Code can't install extensions to a read-only remote"
description: >-
  A read-only (Nix-managed) local extensions directory makes VS Code Remote installs fail with
  EntryWriteLocked. Why it happens, and how to work around it.
date: 2026-06-23
tags: [tooling]
linkedin: 'https://www.linkedin.com/posts/maksim-shcherbo_softwareenginerring-vscode-nix-share-7474974533335101440-vH5q/'
--->
# Why VS Code can't install extensions to a remote when your local ones are read-only

> 📍 **Canonical version: [happygopher.nl/writing/vscode-remote-readonly-extensions](https://happygopher.nl/writing/vscode-remote-readonly-extensions/).** This copy is kept for existing links.

If your local VS Code extensions directory is read-only (common with Nix-managed setups) and you work over
VS Code Remote (Remote-SSH, WSL, Dev Containers, Codespaces), installing an extension onto the remote fails
with a misleading error:

```
Unable to write file '/home/you/.vscode-server/extensions/.<uuid>/package.json'
  (EntryWriteLocked (FileSystemError): Error: EACCES: permission denied,
   open '/home/you/.vscode-server/extensions/.<uuid>/package.json')
    at updateManifestMetadata → extractUserExtension → extractVSIX
```

## TLDR

A read-only local extensions directory (common with Nix) gives you reproducibility and some hardening, and
it breaks installing extensions onto a remote. The ones that fail are the ones you already have installed
locally: VS Code reuses the local copy by zipping it, preserving the 0444 mode bits, so the files it unpacks
on the server are read-only and the install can't rewrite them (`EntryWriteLocked` / EACCES). This is a known
VS Code bug ([#151543][issue]) with no merged fix. Install remote extensions with `code --install-extension`
from the integrated terminal instead.

_Verified against VS Code 1.124.2, June 2026. The bug is still open ([#151543][issue]); check it before
assuming this is current._

## Background: how VS Code installs extensions on a remote

VS Code Remote splits the editor in two. The window runs on your local machine. A second copy, the VS Code
Server, runs on the remote and hosts the extensions that need to be there. Each extension declares where it
runs: UI extensions (themes, keymaps) stay local; workspace extensions (language servers, linters,
formatters) run on the remote, in `~/.vscode-server/extensions`.

How an extension reaches the remote depends on whether you already have it installed locally. The bug is in
that branch.

## When the local extensions directory is read-only

The trigger is a read-only local extensions directory. The most common source is managing extensions
declaratively with Nix and home-manager (`programs.vscode.profiles.default.extensions`), which makes
`~/.vscode/extensions` a read-only tree of `/nix/store` symlinks. A read-only `--extensions-dir` or an
immutable editor image produces the same result.

This is usually deliberate, for two reasons:

- Reproducibility. The editor is rebuilt from a config file, with no drift and no unrequested auto-updates.
- Hardening. VS Code's extension security model is weak: extensions are ordinary Node code with broad
  filesystem access. A read-only extensions directory prevents an installed extension from rewriting itself
  or its neighbours on disk. It does not prevent a malicious extension from writing elsewhere, but it removes
  one foothold.

VS Code assumes this directory is writable. That assumption is the bug.

## The mechanism

### EntryWriteLocked means the file is not writable

Despite the name, this is a permission failure, not a lock. From [`diskFileSystemProvider.ts`][src-dfsp],
after a write returns EACCES:

```ts
const { stat } = await SymlinkSupport.stat(this.toFilePath(resource));
// 0o200 = writable by owner
if (!(stat.mode & 0o200)) {
  writeError = createFileSystemProviderError(
    error,
    FileSystemProviderErrorCode.FileWriteLocked,
  );
}
```

`FileWriteLocked` prints as `EntryWriteLocked`. The write returned EACCES, and the stat showed no owner-write
bit. The `package.json` being written was mode 0444.

### The file is 0444 because it came from your read-only copy

When you install an extension you already have locally, VS Code can reuse that local copy instead of
downloading a fresh one. It zips your local files and sends the VSIX to the remote
([`extensionsWorkbenchService.installInServer`][src-iis]):

```ts
// local = your read-only 0444 copy
const vsix = await this.extensionManagementService.zip(local);
return await server.extensionManagementService.install(vsix);
```

[`zip()`][src-zipfn] copies each source file's mode into the VSIX entry, and extraction restores it
([`modeFromEntry`][src-zipmode] in `zip.ts`). Nothing in between makes the files writable:

```ts
// modeFromEntry(entry) derives the file mode from the zip entry. 0o100644
// is only a fallback for when the entry carries no stored mode; it then
// masks attr down and returns it — your entry stores 0444, so it's 0o444.
const attr = entry.externalFileAttributes >> 16 || 33188;

// the caller writes the file with exactly that mode (0o444):
const mode = modeFromEntry(entry);
istream = createWriteStream(targetFileName, { mode });
```

A 0444 file in your local copy becomes a 0444 entry in the VSIX, then a 0444 file on the remote. The next
install step, `updateManifestMetadata`, opens that `package.json` to add an `__metadata` block and gets
EACCES.

A snapshot of the staging directory during a failing install shows every file read-only:

```
.<uuid>/                     755
.<uuid>/package.json         444
.<uuid>/readme.md            444
```

The same extension installed from the command line extracts at 0644.

### Why some extensions still work

The failure requires VS Code to reuse your local read-only copy. When the install goes through a fresh
marketplace download instead, the files are 0644 and it succeeds:

| extension you install | have it locally? | path taken                            | result         |
| --------------------- | ---------------- | ------------------------------------- | -------------- |
| already installed     | yes (read-only)  | zips and ships the 0444 copy          | fails (remote) |
| not installed         | no               | fresh download from the marketplace   | works          |
| a UI-kind extension   | n/a              | installs into the local read-only dir | fails (local)  |

The server log shows which path ran. Marketplace installs carry options such as `pinned` and
`clientTargetPlatform`; the failing path is bare (`{"isBuiltin":false,"isApplicationScoped":false}`), which
is what installing from a VSIX file logs. The marketplace download is writable, so the 0444 came from your
local copy, not the publisher. `zipinfo` confirms it:

```
$ zipinfo even-better-toml.vsix extension/package.json
-rw-r--r--  ... extension/package.json
```

## What it is not

Plausible explanations, all wrong:

- The remote's permissions. `~/.vscode-server/extensions` is writable; installing the same extension from
  the command line into it succeeds.
- umask. The server runs umask 0022, but the 0444 is an explicit `mode` passed on create, which umask can
  only narrow.
- Concurrency. Several parallel installs succeed; the `EntryWriteLocked` name is misleading.
- A platform-specific extension. The deciding factor is whether the extension is installed locally, not its
  architecture. Platform-specific extensions can avoid the bug because a platform mismatch forces a fresh
  download instead of reusing the local copy.

For UI-kind extensions the error appears on the local machine instead of the remote, so the path is the local
one (`/Users/...` on macOS, `/home/...` on Linux):
`mkdir '/Users/you/.vscode/extensions/.<uuid>' EACCES`. Reading only the remote server logs, you never see
it.

## A known, unfixed VS Code bug

This is [microsoft/vscode#151543][issue]: "read-only `--extensions-dir` (common in Nix deployments)… copies
preserving 0444… modify package.json fails." Status: open, Backlog, labelled `team-low-hanging`. The one fix
attempt, [PR #242256][pr], added the missing write bits when the VSIX is created:

```ts
// OR-in user+group write
mode: stat.mode | 0o220,
```

It was closed without merging ("still under discussion… closing this PR until then"), so it will not resolve
on its own.

The same failure is filed on the Nix side as [home-manager#7188][hm-issue]: extensions installed through
home-manager's `profiles` fail on the remote with this exact error, while the FHS build (`vscode-fhs`)
installs them. The difference is the read-only store. `vscode-fhs` keeps a writable extensions directory, so
the copied files come out writable.

## Solutions

| approach                                                | reliable?                    | keeps the read-only directory? | cost                                          |
| ------------------------------------------------------- | ---------------------------- | ------------------------------ | --------------------------------------------- |
| reactive `chmod` watcher (inotify/poll)                 | no, races the metadata write | yes                            | a background service, and unreliable          |
| `bindfs` presenting the remote server dir as writable   | yes                          | yes                            | a maintained FUSE mount                       |
| `CAP_DAC_OVERRIDE` on the server's `node`               | yes                          | yes                            | broad capability, re-apply per server version |
| make the local extensions directory writable            | yes, removes the 0444 source | no                             | gives up reproducibility and hardening        |
| `code --install-extension` from the integrated terminal | yes, forces a fresh download | yes                            | one line per extension                        |

The reactive watcher is unreliable. The chmod must land between each file's creation and the metadata write,
and it loses that race for any extension with more than a few files: a small one installs, a larger one does
not. Making the directory writable works but gives up the reproducibility and hardening that motivated the
read-only setup.

## The fix: code --install-extension

Install remote extensions from the integrated terminal connected to the remote:

```sh
code --install-extension rust-lang.rust-analyzer \
     --install-extension tamasfe.even-better-toml \
     --install-extension jnoortheen.nix-ide
```

This only works in the remote integrated terminal, where `code` is the remote command-line tool: it talks to
the running server and installs each extension with a fresh download to the writable remote directory, so the
reuse path is never taken. Run the same command in a local terminal and `code` is your local binary, which
installs the extension locally instead. Installed extensions persist in `~/.vscode-server/extensions` on the
remote, so this is a one-time action, not something you repeat per connection.

Avoid the GUI "Install in <remote>" button for extensions you already have locally. The local editor stays
read-only and reproducible. The remote does not: these extensions are installed imperatively, outside the Nix
manifest, so you reinstall them when the box is rebuilt. To keep the remote reproducible, put the
`code --install-extension` lines in a script the box runs at setup (a postCreate hook, or a small unit in its
config).

## References

- [microsoft/vscode#151543][issue]: read-only `--extensions-dir` breaks remote extension installation
- [microsoft/vscode#242256][pr]: the abandoned fix (`stat.mode | 0o220`)
- [nix-community/home-manager#7188][hm-issue]: the same error reported for home-manager `profiles`
- home-manager [`programs.vscode.mutableExtensionsDir`][hm]: the option that makes the local directory
  writable (mutually exclusive with `profiles`)
- Source, pinned to VS Code [commit `6928394`][src-commit] (1.124.2):
  [`FileWriteLocked`][src-dfsp] · [`zip()`][src-zipfn] · [`modeFromEntry`][src-zipmode] ·
  [`installInServer`][src-iis] · [`extractUserExtension`][src-extract] ·
  [`updateManifestMetadata`][src-updmeta]

[issue]: https://github.com/microsoft/vscode/issues/151543
[pr]: https://github.com/microsoft/vscode/pull/242256
[hm-issue]: https://github.com/nix-community/home-manager/issues/7188
[hm]: https://mynixos.com/home-manager/option/programs.vscode.mutableExtensionsDir
[src-commit]: https://github.com/microsoft/vscode/tree/6928394f91b684055b873eecb8bc281365131f1c
[src-dfsp]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/platform/files/node/diskFileSystemProvider.ts#L877-L890
[src-zipmode]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/base/node/zip.ts#L52-L58
[src-zipfn]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/platform/extensionManagement/node/extensionManagementService.ts#L116-L127
[src-iis]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/workbench/contrib/extensions/browser/extensionsWorkbenchService.ts#L2779-L2799
[src-extract]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/platform/extensionManagement/node/extensionManagementService.ts#L620-L678
[src-updmeta]: https://github.com/microsoft/vscode/blob/6928394f91b684055b873eecb8bc281365131f1c/src/vs/platform/extensionManagement/common/extensionsScannerService.ts#L303-L310
