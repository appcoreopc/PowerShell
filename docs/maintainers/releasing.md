# PowerShell Core Releasing Process

## Release Steps

> Note: Step 2, 3 and 4 can be done in parallel. Step 5 and 6 can be done in parallel.

1. Create a branch named `release` in `PowerShell/PowerShell` repository. All release related changes should happen in this branch.
1. Prepare packages
   - [Build release packages](#building-packages)
   - Sign the MSI packages and DEB/RPM packages.
   - Install and verify the packages.
1. Update documentation, scripts and Dockerfiles
   - Summarize the change log for the release. It should be reviewed by PM(s) to make it more user-friendly.
   - Update [CHANGELOG.md](../../CHANGELOG.md) with the finalized change log draft.
   - Update other documents and scripts to use the new package names and links.
1. Verify the release Dockerfiles.
1. Publish Linux packages to Microsoft YUM/APT repositories.
1. [Create NuGet packages](#nuget-packages) and publish them to [powershell-core feed][ps-core-feed].
1. [Create the release tag](#release-tag) and push the tag to `PowerShell/PowerShell` repository.
1. Merge the `release` branch to `master` and delete the `release` branch.
1. Publish the release in Github.
1. Trigger the release docker builds for Linux and Windows container images.
   - Linux: push a branch named `docker` to `powershell/powershell` repository to trigger the build at [powershell docker hub](https://hub.docker.com/r/microsoft/powershell/builds/).
     Delete the `docker` branch once the builds succeed.
   - Windows: queue a new build in `PowerShell Windows Docker Build` on VSTS.
1. Verify the generated docker container images.
1. [Update the homebrew formula](#homebrew) for the OSX package.
   This task usually will be taken care of by the community,
   so we can wait for one day or two and see if the homebrew formula has already been updated,
   and only do the update if it hasn't.

## Building Packages

> Note: Linux and Windows packages are taken care of by our release build pipeline in VSTS,
while the OSX package needs to be built separately on a macOS.

The release build should be started based on the `release` branch.
The release Git tag won't be created until all release preparation tasks are done,
so the to-be-used release tag should be passed to the release build as an argument.

> When creating the packages, please ensure that the file path does not contain user names.
That is, clone to `/PowerShell` on Unix, and `C:\PowerShell` for Windows.
The debug symbols include the absolute path to the sources when built,
so it should appear `/PowerShell/src/powershell/System.Management.Automation`,
not `/home/username/src/PowerShell/...`.

### Packaging Overview

The `build.psm1` module contains a `Start-PSPackage` function to build packages.
It **requires** that PowerShell Core has been built via `Start-PSBuild`.

#### Windows

The `Start-PSPackage` function delegates to `New-MSIPackage` which creates a Windows Installer Package of PowerShell.
The packages *must* be published in release mode,
so make sure `-Configuration Release` is specified when running `Start-PSBuild`.

It uses the Windows Installer XML Toolset (WiX) to generate a MSI package,
which copies the output of the published PowerShell files to a version-specific folder in Program Files,
and installs a shortcut in the Start Menu.
It can be uninstalled through `Programs and Features`.

Note that PowerShell is always self-contained, thus using it does not require installing it.
The output of `Start-PSBuild` includes a `powershell.exe` executable which can simply be launched.

#### Linux / macOS

The `Start-PSPackage` function delegates to `New-UnixPackage`.
It relies on the [Effing Package Management][fpm] project,
which makes building packages for any (non-Windows) platform a breeze.
Similarly, the PowerShell man-page is generated from the Markdown-like file
[`assets/powershell.1.ronn`][man] using [Ronn][].
The function `Start-PSBootstrap -Package` will install both these tools.

To modify any property of the packages, edit the `New-UnixPackage` function.
Please also refer to the function for details on the package properties
(such as the description, maintainer, vendor, URL,
license, category, dependencies, and file layout).

> Note that the only configuration on Linux and macOS is `Linux`,
> which is release (i.e. not debug) configuration.

To support side-by-side Unix packages, we use the following design:

We will maintain a `powershell` package
which owns the `/usr/bin/powershell` symlink,
is the latest version, and is upgradeable.
This is the only package named `powershell`
and similarly is the only package owning any symlinks,
executables, or man-pages named `powershell`.
Until we have a package repository,
this package will contain actual PowerShell bits
(i.e. it is not a meta-package).
These bits are installed to `/opt/microsoft/powershell/6.0.0-alpha.8/`,
where the version will change with each update
(and is the pre-release version).
On macOS, the prefix is `/usr/local`,
instead of `/opt/microsoft` because it is derived from BSD.

> When we have access to package repositories where dependencies can be properly resolved,
> this `powershell` package can become a meta-package which auto-installs the latest package,
> and so only owns the symlink.

For explicitly versioned packages, say for PowerShell 6.0,
we will maintain separate packages named in the form `powershell6.0`,
which owns the binary `powershell6.0`, the symlink `powershell6.0`,
the man-page `powershell6.0`,
and is installed to `/opt/microsoft/powershell/6.0/`.
Specifically this package owns nothing named `powershell`,
as only the `powershell` package owns those files.
This package is upgradeable, but should only be updated with hot-fixes.
This is a necessary consequence of Unix package managers,
as files among packages *cannot* conflict.
From a user-experience perspective,
if the user requires a specific version of PowerShell,
they should not be required to use an absolute path,
and instead should be given a binary with the version in the name.
This pattern is followed by many other languages
(Python being the most obvious example).
This same pattern can be followed for versions 6.1, 7.0, etc.,
and can be used for patch version (e.g. 6.0.1).
Use `Start-PSPackage -Name powershell6.0` to generate
the versioned `powershell6.0` package.
Without `-Name` specified, the primary `powershell`
package will instead be created.

[fpm]: https://github.com/jordansissel/fpm
[man]: ../../assets/powershell.1.ronn
[ronn]: https://github.com/rtomayko/ronn

### Build and Packaging Examples

On macOS or a supported Linux distro, run the following commands:

```powershell
# Install dependencies
Start-PSBootstrap -Package

# Build for v6.0.0-beta.1 release
Start-PSBuild -Clean -Crossgen -PSModuleRestore -ReleaseTag v6.0.0-beta.1

# Create package for v6.0.0-beta.1 release
Start-PSPackage -ReleaseTag v6.0.0-beta.1
```

On Windows, the `-Runtime` parameter should be specified for `Start-PSBuild` to indicate what version of OS the build is targeting.

```powershell
# Install dependencies
Start-PSBootstrap -Package

# Build for v6.0.0-beta.1 release targeting Windows 10 and Server 2016
Start-PSBuild -Clean -CrossGen -PSModuleRestore -Runtime win10-x64 -Configuration Release -ReleaseTag v6.0.0-beta.1
```

If the package is targeting a downlevel Windows (not Windows 10 or Server 2016),
the `-WindowsDownLevel` parameter should be specified for `Start-PSPackage`.
Otherwise, the `-WindowsDownLevel` parameter should be left out.

```powershell
# Create packages for v6.0.0-beta.1 release targeting Windows 10 and Server 2016.
# When creating packages for downlevel Windows, such as Windows 8.1 or Server 2012R2,
# the parameter '-WindowsDownLevel' must be specified.
Start-PSPackage -Type msi -ReleaseTag v6.0.0-beta.1 <# -WindowsDownLevel win81-x64 #>
Start-PSPackage -Type zip -ReleaseTag v6.0.0-beta.1 <# -WindowsDownLevel win81-x64 #>
```

## NuGet Packages

In the `release` branch, run `Publish-NuGetFeed` to generate PowerShell NuGet packages:

```powershell
# Assume the to-be-used release tag is 'v6.0.0-beta.1'
$VersionSuffix = ("v6.0.0-beta.1" -split '-')[-1]

# Generate NuGet packages
Publish-NuGetFeed -VersionSuffix $VersionSuffix
```

PowerShell NuGet packages and the corresponding symbol packages will be generated at `PowerShell/nuget-artifacts` by default.
Currently the NuGet packages published to [powershell-core feed][ps-core-feed] only contain assemblies built for Windows.
Maintainers are working on including the assemblies built for non-Windows platforms.

[ps-core-feed]: https://powershell.myget.org/gallery/powershell-core

## Release Tag

PowerShell releases use [Semantic Versioning][semver].
Until we hit 6.0, each sprint results in a bump to the build number,
so `v6.0.0-alpha.7` goes to `v6.0.0-alpha.8`.

When a particular commit is chosen as a release,
we create an [annotated tag][tag] that names the release.
An annotated tag has a message (like a commit),
and is *not* the same as a lightweight tag.
Create one with `git tag -a v6.0.0-alpha.7 -m <message-here>`,
and use the release change logs as the message.
Our convention is to prepend the `v` to the semantic version.
The summary (first line) of the annotated tag message should be the full release title,
e.g. 'v6.0.0-alpha.7 release of PowerShellCore'.

When the annotated tag is finalized, push it with `git push --tags`.
GitHub will see the tag and present it as an option when creating a new [release][].
Start the release, use the annotated tag's summary as the title,
and save the release as a draft while you upload the binary packages.

[semver]: http://semver.org/
[tag]: https://git-scm.com/book/en/v2/Git-Basics-Tagging
[release]: https://help.github.com/articles/creating-releases/

## Homebrew

After the release, you can update homebrew formula.

On macOS:

1. Make sure that you have [homebrew cask](https://caskroom.github.io/).
1. `brew update`
1. `cd /usr/local/Homebrew/Library/Taps/caskroom/homebrew-cask/Casks`
1. Edit `./powershell.rb`, reference [file history](https://github.com/vors/homebrew-cask/commits/master/Casks/powershell.rb) for the guidelines:
    1. Update `version`
    1. Update `sha256` to the checksum of produced `.pkg` (note lower-case string for the consistent style)
    1. Update `checkpoint` value. To do that run `brew cask _appcast_checkpoint --calculate 'https://github.com/PowerShell/PowerShell/releases.atom'`
1. `brew cask style --fix ./powershell.rb`, make sure there are no errors
1. `brew cask audit --download ./powershell.rb`, make sure there are no errors
1. `brew cask reinstall powershell`, make sure that powershell was updates successfully
1. Commit your changes, send a PR to [homebrew-cask](https://github.com/caskroom/homebrew-cask)
