# Maintaining the osxcross and cctools-port forks

This is the working procedure for the two forks of the upstream projects of the same names. Each fork adds a release pipeline: a tag push packages the SDKs of every Xcode that GitHub's macOS images carry, builds Apple Clang for Ubuntu, builds toolchains on top of them, and publishes everything as assets of a GitHub release. The work is to keep that in step with upstream and with the runner images, and to release.

Use it when upstream osxcross or cctools-port publishes a release, when GitHub retires a runner image or Ubuntu ships a release, when Apple Clang moves, and when a release run fails. The method in one sentence: merge upstream, adapt the workflows, prove the result on a precheck branch, then release by tag push and watch CI.

Paths and names that belong to one installation are placeholders: `<account>` is the GitHub account that hosts the forks, `<osxcross>` and `<cctools-port>` are the local checkouts, `<tag>` is the tag of the release being made.

## Contents

1. [Scope and deliverables](#1-scope-and-deliverables)
2. [Rules](#2-rules)
3. [Workflow](#3-workflow)
4. [How to reference](#4-how-to-reference)
5. [What to reference](#5-what-to-reference)
6. [Background: the pipelines](#6-background-the-pipelines)
7. [Troubleshooting](#7-troubleshooting)

## 1. Scope and deliverables

In scope:

- the branch of each fork that a release is cut from: `main` for osxcross, the newest version branch for cctools-port;
- the fork's own files: `.github/workflows/release.yml`, `sdk.yml` and `precheck.yml` in both repositories, `precheck-clang.yml` and `probe-images.yml` in osxcross, and `tools/gen_sdk_package.sh` of cctools-port, which is the fork's copy of a script that upstream has rewritten since;
- the fork's fixes inside upstream files: `build.sh`, `build_clang.sh` and `tools/gen_sdk_package.sh` of osxcross, `usage_examples/ios_toolchain/build.sh` of cctools-port.

Deliverables:

- the pushed branch and a tag with a complete release;
- a report in four parts: what changed upstream and what is worth having, the failures of CI and their fixes, the asset count of the release against the previous one, and what was left open.

Related and not covered here: the xcode-sdk fork packages SDKs with a copy of `sdk.yml` and needs the same changes whenever `sdk.yml` changes; the projects that download the assets of these releases (section 4.4).

## 2. Rules

1. **A release is a tag push.** `release.yml` runs on `v*`, creates the release, and uploads to it. There are no release notes.
2. **Prove it on a precheck branch before the tag.** The release workflow only starts with a tag, and a failed run leaves an incomplete release visible until it is fixed. The precheck costs one to two hours.
3. **A precheck runs what the release would run.** It packages the SDKs with the workflow of the release and builds with what that produced. Names written into a workflow go stale with the runner images.
4. **osxcross first, cctools-port second.** The cctools-port workflow downloads Apple Clang from an osxcross release. Its tag comes once that release has its eight Apple Clang packages; the SDKs and toolchains of osxcross are of no use to it.
5. **Merge upstream, do not rebase.** The fork's history is upstream's history plus a merge commit and a few own commits.
6. **Re-check every assumption of the workflows after a merge.** They are listed in section 3.4. An upstream change can break a pipeline without a merge conflict, and some breaks are silent.
7. **A failure is diagnosed and fixed, not retried blindly.** Cancel the run, push the fix, delete the tag and push it again with the same number. Rerun only what is known to be transient.
8. **Each sync includes an analysis**: what changed upstream, what is worth adopting, whether a new Ubuntu release or a new runner image changes the matrix, whether a newer Apple Clang or a newer cctools exists.
9. **Commit with the configured identity, without trailers.** The subjects follow the history: `fix:`, `chore:`, `ci:`.

## 3. Workflow

| Step | What you do | What you have at the end |
|---|---|---|
| [3.1](#31-prepare) | fetch both remotes, compare with the fork | local equal to the fork |
| [3.2](#32-survey-upstream) | read what changed upstream and on the runners | the list of things to adapt |
| [3.3](#33-merge-upstream) | merge the upstream branch | a merge commit |
| [3.4](#34-adapt-the-workflows) | check each assumption of the workflows | commits `fix:` and `ci:` |
| [3.5](#35-precheck) | push a `precheck/` branch, read the run | a green precheck |
| [3.6](#36-release-osxcross) | push `main`, tag, watch, count assets | a complete osxcross release |
| [3.7](#37-release-cctools-port) | version branch, reference to the new release, tag | a complete cctools-port release |
| [3.8](#38-report) | report | the four parts of section 1 |

### 3.1 Prepare

```sh
cd <osxcross>
git fetch origin --tags
git fetch upstream
git status --short
git rev-list --left-right --count main...origin/main
git rev-list --left-right --count main...upstream/master
```

The local branch must be equal to `origin/main` before anything else. When the local branch has strayed, keep it under a backup name (`git branch main-backup-local`) and reset it to `origin/main`.

The remotes: `origin` is the fork, `upstream` is `https://github.com/tpoechtrager/osxcross.git` and `https://github.com/tpoechtrager/cctools-port.git`.

cctools-port has one branch per cctools version (`1030.6.3-ld64-956.6`, `1035.1.102-ld64-957.1`, ...). Its upstream has no usable `master`; every command names the version branch.

Work on a branch in a second worktree, so that the checkout stays on the release branch:

```sh
git worktree add <worktree> -b precheck/<topic> main
```

### 3.2 Survey upstream

```sh
git log --oneline main..upstream/master
git diff --stat $(git merge-base main upstream/master) upstream/master
git show upstream/master:CHANGELOG.md | head -60
```

Read for four things:

- **Build scripts and wrapper.** These run in the pipeline. A rewrite of the wrapper is a reason to precheck, not to hold back. Note every commit that touches a file the pipeline runs: `build.sh`, `build_clang.sh`, `tools/tools.sh`, `tools/gen_sdk_package.sh`.
- **Versions.** `build.sh` names the cctools and ld64 versions of its `stable` and `latest` flavors, `build_clang.sh` the Apple Clang versions it knows and the LLVM release it defaults to. A new cctools version usually arrives as a new branch of cctools-port first.
- **What is worth having.** The changelog says which SDKs, compilers and features arrived.
- **Requirements of the build host.** Since v3.1.2 the build scripts need GNU sed on macOS hosts. CI builds on Linux, so this matters for users only.

Then the two sources outside the repositories (section 4):

- the runner images: which macOS images exist, which Xcodes they carry, which labels were retired;
- Ubuntu: whether a release is missing from the matrix. The matrix covers the LTS releases.

Done when the list of things to adapt is written down.

### 3.3 Merge upstream

```sh
git merge upstream/master --no-edit             # osxcross
git merge upstream/<version-branch> --no-edit   # cctools-port
```

Then look at what the fork still changes on top of upstream; this is the part a merge can silently mangle:

```sh
git diff --stat upstream/master                      # the whole delta
git diff upstream/master -- build.sh build_clang.sh tools/gen_sdk_package.sh
```

It must be exactly the fork's fixes (section 6.5). An automatic merge can place a hunk in a function that upstream restructured.

### 3.4 Adapt the workflows

Check each of these after a merge.

| Assumption | Check |
|---|---|
| The scripts still read the variables the workflows set: `UNATTENDED`, `SDK_VERSION`, `IPHONEOS_DEPLOYMENT_TARGET`, `CLANG_VERSION`, `ENABLE_CLANG_INSTALL`, `PORTABLE`, `INSTALLPREFIX`, `JOBS` | `grep -c <name> build.sh build_clang.sh build_apple_clang.sh`; none may be 0 |
| `sdk.yml` inserts a line into `tools/gen_sdk_package.sh` with `sed` | the pattern of the `sed` matches a line of the script; a pattern that no longer matches is silent |
| The packaging script of osxcross still finishes on an SDK that contains a link cycle | `grep -c "directory causes a cycle" tools/gen_sdk_package.sh` must be 1 |
| The runner labels of `sdk.yml` exist | a retired label cancels the job within a second; section 4.2 |
| The package names of the `apt-get install` lines exist in the oldest Ubuntu of the matrix | `lzma-dev` was dropped by 26.04; verify against `http://archive.ubuntu.com/ubuntu/dists/<codename>/main/binary-amd64/Packages.gz` |
| `release.yml` builds the newest Apple Clang the script knows | `grep -n "CLANG_LLVM_PKG" build_clang.sh`; a new version changes `CLANG_VERSION` and the `apple-clang-<n>-` prefix of the assets |
| The cctools version of the fork matches what upstream osxcross uses for its `latest` flavor | `grep -n "LATEST_CCTOOLS_VERSION" build.sh` |
| `npm install @actions/artifact@2` at every call site of the SDK jobs | without the pin the `Collect SDKs` job fails in seconds |
| `PREVIOUS_RELEASE` in `precheck.yml` of osxcross is the last complete release | its SDKs and Apple Clang feed the builds |
| The SDK versions of the build matrix are the ones `Collect SDKs` uploaded | `grep -c listArtifacts .github/workflows/sdk.yml` must print 1; the versions come from the upload loop, and `Release SDKs` downloads that list (section 6.3) |

The fork's `sed` in `sdk.yml` is:

```sh
sed 's|^for SDK in .*; do$|&\n[ -f "$WDIR/$SDK$SDK_EXT" ] \&\& continue|' tools/gen_sdk_package.sh
```

It keeps an SDK that is already packaged, so that the newest Xcode on the runner wins for an SDK that several Xcodes ship. Match the loop header by shape, never by its exact text: upstream rewrites it between versions, and a pattern that no longer matches stops skipping without a word.

The packaging script of osxcross can be tried on any Mac that has Xcode installed, which is faster than a round trip through the macOS runners when the fix is in that script:

```sh
mkdir <dir> && cd <dir>
sed 's|^for SDK in .*; do$|&\n[ -f "$WDIR/$SDK$SDK_EXT" ] \&\& continue|' \
  <osxcross>/tools/gen_sdk_package.sh | \
  XCODEDIR=/Applications/Xcode.app SDK_COMPRESSOR=xz bash
ls -lh <dir>
```

It takes two to four minutes per SDK and leaves 90 to 110 MB per SDK in `<dir>`. To prove that a fix changes nothing else, package the same SDK with the script before the change (`git show <rev>:tools/gen_sdk_package.sh`) and compare the two listings of `tar -tvf`.

Done when every row of the table has an answer and the changes are committed.

### 3.5 Precheck

`release.yml` starts only with a tag. Two workflows run the parts of a release that fail most often, on a branch, and publish nothing. Both trigger on a push to a branch under `precheck/`. (They are bound to a push because GitHub accepts `workflow_dispatch` only for a workflow that exists on the default branch.)

```sh
git push -u origin precheck/<topic>
sleep 180
gh run list --repo <account>/osxcross --branch precheck/<topic> --limit 3
gh run view <run-id> --repo <account>/osxcross --json jobs
```

`precheck.yml`, in both repositories:

1. `package_sdks` calls `sdk.yml`, exactly as the release does.
2. `select_sdks` sorts the versions that the packaging produced and picks the oldest, a middle and the newest device SDK, and for cctools-port also the oldest simulator SDK.
3. The build job runs four combinations of the release matrix with them: the oldest SDK on Ubuntu 24.04 for x86_64, the middle one on 26.04 for aarch64, the newest on 20.04 and on 24.04 for aarch64.

osxcross gets Apple Clang from the last complete release; `PREVIOUS_RELEASE` at the top of its workflow names it. Bump it after each release. The osxcross build then compiles and links a program for two targets with the toolchain it built and lists its dynamic libraries with the installed `otool`; the ioscross build has a compile-and-link check in its build script.

`precheck-clang.yml`, in osxcross only, builds Apple Clang for Ubuntu 20.04 on aarch64 and 26.04 on x86_64, compiles an object for both targets and prints the versions of the compiler and of `ld64.lld`. It takes one and a half to two and a half hours, and it runs only when one of its inputs changes (`build_apple_clang.sh`, `build_clang.sh`, `tools/`), because the build takes that long whatever changed.

The run takes one to two hours for osxcross and half an hour to an hour for cctools-port: the macOS SDK packaging takes 8 to 95 minutes per image, the iOS SDK packaging 4 to 10, the osxcross builds 8 to 15 minutes each, and the ioscross builds 8 to 14 for a device SDK and 13 to 25 for a simulator SDK, where one job builds two toolchains. A job that waits for a macOS runner adds up to twenty minutes (section 7). Read the job list three minutes after the push. Most failures happen within seconds to three minutes, and a run with `fail-fast: false` stays `in_progress` for hours after its first job has failed. The build jobs appear only after the packaging; until then their absence is no finding. Wait for completion with a loop on the status, then delete the branch.

The Linux runners are never the ones that wait. To try the build with an SDK that a packaging job already produced, while another image is still waiting for a runner, push a throwaway workflow on a branch under `probe/`: the build job of the precheck with a fixed matrix, and the SDK fetched from the run that packaged it.

```yaml
      - uses: actions/download-artifact@v4
        with:
          name: <artifact>            # sdk-<runner-image> before Collect SDKs ran
          run-id: <run-id>
          github-token: ${{ github.token }}
          path: tarballs
```

A change to a workflow after its precheck passed is tested by the step it touches: copy that step into a workflow on a `probe/` branch and give it what it reads, such as a few small files uploaded as artifacts under the names it expects.

### 3.6 Release osxcross

Fast-forward `main` to the branch that passed the precheck, remove the branch, then tag. Tags are lightweight and take the next free number: `v0.2.<n>` for osxcross, `v0.1.<n>` for cctools-port.

```sh
cd <osxcross>
git checkout main
git merge --ff-only precheck/<topic>
git push origin main
git push origin --delete precheck/<topic>
git tag <tag>
git push origin <tag>
sleep 180
gh run list --repo <account>/osxcross --limit 3
gh run view <run-id> --repo <account>/osxcross --json jobs
```

The jobs run in this order: `Package SDK` on the macOS images (eight to 95 minutes per image, in parallel with Apple Clang), `Build Apple Clang` (8 jobs, one and a half to two and a half hours), `Release SDKs`, then `Build osxcross` (one job per SDK and Ubuntu image, 8 to 15 minutes each). The release fills up as jobs finish; it is complete when the run is. The whole run takes four to six hours, and a packaging job that waits for a runner adds its waiting time (section 7).

```sh
gh release view <tag> --repo <account>/osxcross --json assets --jq '.assets | length'
gh release view <tag> --repo <account>/osxcross --json assets --jq '.assets[].name' | grep '\.sdk\.tar\.xz$'
```

Compare the count with the previous release and with the matrix: SDKs plus 8 Apple Clang packages plus (SDKs times 8) toolchains. Compare the list of SDKs with the toolsets of the images (section 4.2); an SDK that disappeared is either on a retired image or was not packaged.

A run that failed:

```sh
gh run cancel <run-id>
# fix, push main, then:
git push origin :refs/tags/<tag>
git tag -d <tag>
git tag <tag>
git push origin <tag>
```

The number stays the same until a release is complete. A failure without a change in the tree (a truncated download, a cancelled runner) takes `gh run rerun <run-id> --failed`, which keeps the jobs that passed.

To read the log of a job of a running run, `gh run view --log-failed` refuses; the API works, and only for a job that has finished:

```sh
TOKEN=$(gh auth token)
curl -sL -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/<account>/osxcross/actions/jobs/<job-id>/logs -o job.log
sed 's/\x1b\[[0-9;]*m//g' job.log | tail -80
```

`<job-id>` is `databaseId` in `gh run view <run-id> --json jobs`.

### 3.7 Release cctools-port

This follows the Apple Clang packages of the new osxcross release, because the workflow downloads them from it. They are uploaded while the release run is still going, as each `Build Apple Clang` job finishes; the SDKs and toolchains can take longer, when an image waits for a runner.

If upstream opened a new version branch, open it in the fork too. The branch is upstream's branch plus the fork's files:

```sh
cd <cctools-port>
git switch -c <new-version-branch> upstream/<new-version-branch>
git checkout <old-version-branch> -- .github/workflows tools/gen_sdk_package.sh
git diff upstream/<old-version-branch> <old-version-branch> -- usage_examples/ios_toolchain/build.sh |
  git apply --3way
git commit    # chore: track cctools <version> and ld64 <version>
```

The existing version branch gets a plain merge of its upstream branch, so that it keeps receiving fixes, and the same commits for the workflows as the new branch (`git cherry-pick`); a tag pushed there must still work.

Point the workflow at the new osxcross release: the URL in `release.yml` and in `precheck.yml` contains `osxcross/releases/download/<osxcross-tag>/`; commit the new number as `chore: bump clang`. Precheck as in section 3.5 on the new version branch, then fast-forward the version branch to it and tag:

```sh
git checkout <version-branch>
git merge --ff-only precheck/<topic>
git push origin <version-branch>
git push origin --delete precheck/<topic>
git tag <tag>
git push origin <tag>
```

Check the new URL before the tag: the eight Apple Clang packages of the osxcross release must answer `200`. `<n>` is the Apple Clang version, the number in `CLANG_VERSION`.

```sh
for v in 20.04 22.04 24.04 26.04; do
  for a in x86_64 aarch64; do
    curl -sIL -o /dev/null -w '%{http_code} ' "https://github.com/<account>/osxcross/releases/download/<osxcross-tag>/apple-clang-<n>-linux-$a-gnu-ubuntu-$v.tar.gz"
  done
done
echo
```

When the release is complete, the new version branch becomes the default branch of the repository (`gh repo edit <account>/cctools-port --default-branch <version-branch>`).

The run takes three to four hours, much of it queued (section 4.2). Each `Build ioscross` job builds the toolchain for one SDK, one Ubuntu image and one architecture; for a simulator SDK it builds two toolchains. The asset count is one SDK archive per SDK plus eight toolchains per device SDK (four Ubuntu images on two host architectures) and sixteen per simulator SDK.

### 3.8 Report

Four parts: what changed upstream and what is worth having; the failures of CI and their fixes; the asset count against the previous release, with the SDKs that appeared or disappeared; what was left open.

## 4. How to reference

### 4.1 Upstream

The upstream repository is the reference for what the toolchain does; its changelog is the shortest path to what changed. Read the changelog for versions, the build scripts for what the pipeline calls, and the wrapper directory only when the precheck fails in the wrapper.

Upstream's defaults are worth comparing with the fork's: which cctools it uses for which flavor, which Apple Clang it defaults to, which SDKs it supports. Where upstream and the fork differ, upstream wins unless the fork's difference is a fix that upstream lacks.

When a job fails inside a script that the fork does not change, read the history of that file before changing anything:

```sh
git log --oneline <previous-merge-base>..HEAD -- <file>
git diff <previous-merge-base> HEAD -- <file>
```

A line that upstream removed, such as a `|| true` that swallowed an exit status, is a regression upstream and will be in every checkout of that version. Fix it in the fork's tree with the smallest change that restores the old result, and keep the old behaviour as the measure: the fix is right when the output is the one the previous release produced.

### 4.2 Runner images

The SDK coverage of a release is the set of Xcodes on GitHub's macOS images, and the images change without a notice in either repository. The table of images with their labels is in the README of `actions/runner-images`; the Xcodes of each image are in its toolset (`images/macos/toolsets/toolset-<n>.json`, key `xcode`):

```sh
curl -s https://raw.githubusercontent.com/actions/runner-images/main/images/macos/toolsets/toolset-15.json
```

The arm64 macOS images, with the Xcodes they carried in November 2026:

| Label | Image | Xcodes |
|---|---|---|
| `macos-15` | `macos-15-arm64`, macOS 15 | 16.0 to 16.4, 26.0.1 to 26.3 |
| `macos-26` | `macos-26-arm64`, macOS 26 | 26.0.1 to 26.6 |
| `xcode-27` | `xcode-27-arm64`, macOS 27 | 27.0 to 27.2 |

Check before every release: the labels of `sdk.yml`, and the Xcodes of the images. An SDK is packageable only while some image carries an Xcode that ships it: all macOS 14 images were retired on 2 November 2026, and the macOS 14 and iOS 17 SDKs have not been packageable since. A release that advertises such an SDK would be wrong, and the previous releases keep theirs.

The arm64 macOS runners are scarce. A job can wait fifteen to twenty minutes and then end as `cancelled` with `The job was not acquired by Runner of type hosted even after multiple attempts`; the annotations of the job say so (section 7).

Linux runners start at once, but an account runs twenty jobs at a time over all its repositories. Two releases at once share those slots: 304 build jobs of 8 to 25 minutes took three hours side by side.

The toolsets list the same Xcodes for `x64` and `arm64`: the Intel images `macos-15-intel` and `macos-26-intel` start at once and would package the same SDKs. They are other names of the `-large` runners: larger runners are billed for public repositories too, and they need a payment method or a plan with included compute. Whether that is acceptable is a decision for the owner of the account, not for a release procedure. They have packaged nothing so far.

`.github/workflows/probe-images.yml` answers the questions of this section by running instead of reading: which images the account can use, and which Xcodes they carry. It runs on a push to a branch named `probe/images...`, takes a few minutes per image, and prints the architecture, the macOS version, the default Xcode, the list of Xcodes and the free disk space. Its matrix holds the labels of `sdk.yml`; add others on purpose. Every run starts two Intel jobs, a few seconds each on a billed runner, so the trigger stays narrow: a workflow runs again on each push to a branch it matches.

```sh
git switch -c probe/images precheck/<topic>
# edit the matrix of .github/workflows/probe-images.yml if needed, commit, then:
git push -u origin probe/images
git push origin --delete probe/images
```

### 4.3 Versions of the parts

| Part | Where the version comes from | Where to look for a new one |
|---|---|---|
| Apple Clang | `CLANG_VERSION` in `release.yml`; the source archive in `build_clang.sh` | the `apple/stable` and `swiftlang` `stable/*` branches of `llvm-project`; upstream osxcross adds a version to `build_clang.sh` first |
| LLVM (host compiler for `build_clang.sh`) | default in `build_clang.sh` | `https://github.com/llvm/llvm-project/releases` |
| cctools and ld64 | branch name of cctools-port; the flavor tables in `build.sh` | the branches of `tpoechtrager/cctools-port`, one per Xcode release |
| apple-libtapi, apple-libdispatch | pinned in `build.sh` and `usage_examples/ios_toolchain/build.sh` | their repositories; upstream osxcross moves them |
| Ubuntu images | the matrices of `release.yml` | `https://hub.docker.com/_/ubuntu` and the Ubuntu release list |
| CMake | pinned in the workflows | `https://github.com/Kitware/CMake/releases` |

### 4.4 Consumers

The names of the assets are an interface: projects download them by pattern. Two kinds of change reach them:

- a new kind of launcher in a toolchain can match a consumer's glob by accident: the `*-cmake-clang` launchers broke a consumer that matched `{arch}-apple-darwin*-clang`;
- an SDK name can disappear. Since upstream's packaging change of November 2026 there is one archive per SDK directory, under its most specific name: `MacOSX26.0.sdk.tar.xz` is there, the alias `MacOSX26.sdk.tar.xz` is not.

After a release, run the consumers against it.

## 5. What to reference

| What | Where |
|---|---|
| Upstream osxcross, changelog | `https://github.com/tpoechtrager/osxcross`, `CHANGELOG.md` on `master` |
| Upstream cctools-port, one branch per version | `https://github.com/tpoechtrager/cctools-port` |
| Runner images: labels, Xcodes, retirements | `https://github.com/actions/runner-images` (README and `images/macos/toolsets/`) |
| Apple Clang sources | `https://github.com/apple/llvm-project`, `https://github.com/swiftlang/llvm-project` |
| Docker images of Ubuntu | `https://hub.docker.com/_/ubuntu` |
| Ubuntu package index (does a package exist in a release?) | `http://archive.ubuntu.com/ubuntu/dists/<codename>/` |
| Releases of the forks (assets, counts) | `gh release view <tag> --repo <account>/<repo> --json assets` |
| Runs and logs | `gh run list`, `gh run view <id> --json jobs`, the job log endpoint of section 3.6 |
| A job that ended without running (annotations) | `gh api repos/<account>/<repo>/check-runs/<job-id>/annotations` |
| Which runner images the account can use, and their Xcodes | `.github/workflows/probe-images.yml`, on a `probe/images...` branch |

## 6. Background: the pipelines

### 6.1 osxcross `release.yml`

Triggered by a push of a tag `v*`. Four jobs:

1. `package_sdks` calls `sdk.yml` and returns the list of macOS SDK versions.
2. `build_apple_clang`: 8 jobs, Ubuntu 20.04, 22.04, 24.04 and 26.04 on `ubuntu-latest` and `ubuntu-24.04-arm`. Each writes a script, pulls the Ubuntu image and runs the script in the container: `UNATTENDED=1 ENABLE_CLANG_INSTALL=1 PORTABLE=1 JOBS=$(nproc) bash build_apple_clang.sh`, then uploads `apple-clang-<n>-linux-<arch>-gnu-ubuntu-<version>.tar.gz` to the release. The pull has a retry loop of five attempts.
3. `release_sdks` uploads the SDK archives to the release.
4. `build_osxcross`: one job per macOS SDK version and Ubuntu image (8 per SDK). It downloads the SDK and the Apple Clang package, and runs `UNATTENDED=1 SDK_VERSION=<v> JOBS=$(nproc) bash build.sh` in the container. `UNATTENDED=1` selects the `stable` flavor: cctools 986 and ld64 711. The result is `osxcross-<sdk>-linux-<arch>-gnu-ubuntu-<version>.tar.gz`.

### 6.2 cctools-port `release.yml`

Also triggered by `v*`. `package_sdks` packages the iOS SDKs; `build_ioscross` builds one toolchain per SDK, Ubuntu image and architecture with `usage_examples/ios_toolchain/build.sh`. Apple Clang is downloaded inside the job from an osxcross release, which is why the URL of that release is part of this workflow. Ubuntu 20.04 gets `libstdc++-10-dev` first, because Apple Clang 23 needs `<span>`. The assets are `ioscross-<sdk>-<arm64|x86_64>-linux-<arch>-gnu-ubuntu-<version>.tar.gz`.

### 6.3 `sdk.yml`

A reusable workflow (`workflow_call`) with one job per runner image. Each job lists the Xcodes in `/Applications`, runs `tools/gen_sdk_package.sh` once per Xcode, newest first, and uploads the archives as one artifact per image. `Collect SDKs` then downloads them all, uploads each macOS SDK under its version as an artifact of its own (`26-5`, `15-5`, ...), and reports the names it uploaded; these names are the matrix of the build jobs and the list that `Release SDKs` downloads.

The names must come from the upload loop. Listing the artifacts of the run right after uploading them is a race: the listing lags behind the uploads, and a matrix taken from such a listing lacks SDKs with every job green.

The osxcross script makes one archive per SDK directory, under the name with the most version components; the `sed` line of the workflow keeps the first Xcode that shipped an SDK. The cctools-port script is the older one and still packages every name it finds.

### 6.4 Asset names

| Release | Assets |
|---|---|
| osxcross | `apple-clang-<n>-linux-{x86_64,aarch64}-gnu-ubuntu-{20.04,22.04,24.04,26.04}.tar.gz`, `MacOSX<version>.sdk.tar.xz`, `osxcross-<sdk>-linux-<arch>-gnu-ubuntu-<version>.tar.gz` |
| cctools-port | `iPhoneOS<version>.sdk.tar.xz`, `iPhoneSimulator<version>.sdk.tar.xz`, `ioscross-<sdk>-<target>-linux-<arch>-gnu-ubuntu-<version>.tar.gz` |

### 6.5 The fork's fixes inside upstream files

| File | Fix | Failure without it |
|---|---|---|
| osxcross `build_clang.sh` | `#include <cstdlib>` added to `DenseMapInfo.h` for Apple Clang 23 | `there are no arguments to 'abort'` after two minutes of the Clang job |
| osxcross `build_clang.sh` | CMake flags that leave out benchmarks, examples, tests and docs; install with `cmake --install --strip` | larger and slower builds |
| osxcross `build.sh` | `CPPFLAGS="-isystem $(llvm-config --includedir)"` exported before the cctools `configure`, in `build_stable` and `build_latest`, and unset after it | `'llvm-c/Visibility.h' file not found` at five minutes of the osxcross job |
| osxcross `tools/gen_sdk_package.sh` | the copy step reads the diagnostics of `cp` from a file and fails only on those that are not `directory causes a cycle` | `Package SDK` stops at the first SDK of Xcode 26.2 and later, and packages nothing |
| cctools-port `usage_examples/ios_toolchain/build.sh` | `CFLAGS`/`CXXFLAGS` with `-Wno-error=unused-but-set-global` for apple-libdispatch; the same `CPPFLAGS` export | `set but not used [-Werror,-Wunused-but-set-global]` after a minute of the ioscross job |

The `CPPFLAGS` fix became redundant where cctools copies `Visibility.h` itself (upstream PR 192); it is harmless and still needed by the `stable` flavor.

## 7. Troubleshooting

Each entry is a symptom, its cause, and what to do.

**`Collect SDKs` fails in ten seconds with `ERR_PACKAGE_PATH_NOT_EXPORTED`.** An unpinned `npm install @actions/artifact` resolves to version 3 or later. Pin `@actions/artifact@2` at every call site, in both workflows and in every repository that has them.

**`Cannot convert 15-0 to a BigInt` in a step that downloads an artifact.** `@actions/artifact@2` takes the numeric id of an artifact, not its name. List the artifacts and pick the id.

**`Package SDK` fails in its first minutes with `cp: .../Ruby.framework/.../Headers/ruby/ruby: directory causes a cycle`, and no SDK was packaged.** The macOS SDKs since Xcode 26.2 contain a link that points to its own directory. `cp -r` follows links: it leaves that one out, copies everything else, and exits non-zero. Upstream's script runs that copy under `set -e` without a guard, which ends the script. The fork's script tolerates that one diagnostic (section 6.5); after a merge, check that the fix is still there.

**A `Package SDK` job is cancelled within a second, with no step.** The runner label is retired. The annotation says `The configuration ... is not supported`. Replace the label in `sdk.yml` (section 4.2), in both forks.

**A queued macOS job ends as `cancelled` after fifteen to twenty minutes.** `The job was not acquired by Runner of type hosted even after multiple attempts`: the arm64 macOS pool had no capacity. Everything that needs the job is skipped. A single job cannot be re-run while its run is in progress (`The workflow run containing this job is already running`). Two ways out: let the run end and `gh run rerun <run-id> --failed`, which keeps the jobs that passed; or, when the other jobs have run for a few minutes only, `gh run cancel <run-id>` and `gh run rerun <run-id>`, so that the images queue again at once instead of one after the other. Section 4.2 has the alternative.

When the pool stays scarce, a loop does the retrying: `sleep 60` on the status of the run, and `gh run rerun <run-id> --failed` whenever the run has completed and every job that did not succeed is `cancelled` or `skipped`. A job that failed for another reason ends the loop. Stop the loop before cancelling a run on purpose, or it re-runs what was cancelled.

**`Build Apple Clang` fails after two minutes with `DenseMapInfo.h: ... 'abort'`.** The Apple Clang sources call `abort()` without including `<cstdlib>`, and the stage-one build uses the g++ of the container. The fix for version 23 is in `build_clang.sh`; a new Apple Clang version needs the same line for its version number.

**`E: Unable to locate package lzma-dev`.** Ubuntu 26.04 dropped the transitional package. Use `liblzma-dev` in every `apt-get install` list.

**`Build osxcross` fails at five minutes with `'llvm-c/Visibility.h' file not found`.** The cctools `configure` copies `lto.h` from the LLVM it finds and does not add that LLVM's include directory. Export `CPPFLAGS` before `./configure` (section 6.5).

**`Build osxcross` fails with `'stuff/openstep_mach.h' file not found`.** `CPPFLAGS` was passed on the make command line; make splits the value at the space and takes the directory as a target. The variable has to be exported before `configure`.

**A test of a built toolchain fails where the build's own tests passed.** Three things such a test must get right:

- `export PATH=<target-dir>/bin:$PATH` first. The wrapper leaves `PATH` alone and clang looks the linker up there; without it the link goes to the `ld` of the container (`/usr/bin/ld: unrecognised emulation mode: llvm`). The build script sets it for its own tests, and that does not reach the calling shell.
- Call the launchers as the README does: `x86_64-apple-darwin<target>-clang` and `arm64-apple-darwin<target>-clang`, or the shortcuts `o64-clang` and `oa64-clang`. There is no `arm64-clang`. The target in the name cannot be derived from the SDK version: SDK 27.0 installs `...-darwin27-clang`, SDK 27.2 `...-darwin27.2-clang`. Take the names from the installation (`"$TARGET_DIR"/bin/x86_64-apple-darwin*-clang`).
- The `stable` flavor installs the cctools under their target prefix only (`arm64-apple-darwin<version>-otool`); the unprefixed `otool` and `lipo` are links of the `llvm` flavor.

**Many jobs fail in under a minute with `toomanyrequests`.** Anonymous pulls of the Ubuntu image hit the rate limit of Docker Hub. Rerun the failed jobs. The pull retry loop in `release.yml` helps only with transient errors; cctools-port has no loop, and it downloads Apple Clang through a pipe, so a truncated transfer is not retried either.

**`Build ioscross` fails after a minute with `-Werror,-Wunused-but-set-global`.** A new diagnostic of Apple Clang in apple-libdispatch. The flags are in section 6.5.

**`gzip: stdin: unexpected end of file` in one job of a matrix.** A truncated download. `gh run rerun <run-id> --failed`, no change.

**An SDK is missing from the release and no job failed.** Either the `sed` in `sdk.yml` no longer matches (section 3.4), or the Xcode that carried the SDK left the runner images (section 4.2). Counting the assets after a release (section 3.6) shows it.

**`Release SDKs` fails with `not in the artifact listing: <names>`.** The artifact listing did not catch up with the uploads of `Collect SDKs` within a minute. `gh run rerun <run-id> --failed`, no change.

**A release has fewer SDK archives than its predecessor.** An SDK is packaged once, under its most specific name; archives under an alias name are not made any more (section 4.4). Consumers that asked for an alias need the specific name.

**The precheck lists no build jobs.** They follow the packaging. A job of the packaging that failed or was cancelled skips them; look at `package_sdks` first.

**`git status` says the branch is current, but it is not.** It compares with the last fetch. `git fetch origin --tags` first, then count.

**`no matches found: ^upstream/master`.** zsh treats `^` as a glob. Write `main --not upstream/master`, or quote.

**`gh run view --job <id> --log-failed` on a running run.** Logs of a run in progress come through the API only, and only for a job that has finished (section 3.6).

**The run is `in_progress` for hours and a job failed early.** `fail-fast: false` keeps the run alive. Look at the job list, not at the status of the run.

**A search for `error` in a log finds `liberror-perl`.** Read the tail of the log, or the context of `##[error]`.
