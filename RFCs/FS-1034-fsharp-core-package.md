# F# Tooling RFC FS-1034 - FSharp.Core delivery via NuGet packages

**NOTE:** This is a draft and remains to be reviewed by key contributors.  Please contribute to the discussion and submit adjustments to the draft.

The purpose of this RFC document is to develop a shared plan of record for the delivery of FSharp.Core in NuGet package form that takes
into account the needs of F# Core Engineering contributors. This is a long-term
technical plan to converge to an optimal solution that satisfies both users of F# and those delivering tooling for it.

This is not as simple as readers may expect. It is probably not interesting except to those contributing to F# Core Engineering.
Also, this does not cover the actual functionality contained in FSharp.Core.

Discussion thread: https://github.com/fsharp/fslang-design/issues/197

### Background

Historically ``FSharp.Core.dll`` was compiled and delivered by Microsoft as a .NET 4.x binary installed under "Reference Assemblies".  The DLL
was signed and strong-named by Microsoft. Way, way back in time it was called ``fslib.dll``.

Subsequently, ``FSharp.Core.dll`` has been recompiled for "trimmed down" platforms: portable profiles, Xamarin profiles,
.NET Core, .NET Standard and .NET Framework.  There is even a version of FSharp.Core for the "Xamarin iOS TV" profile....
When provided by Microsoft, these were signed, strong-named and installed under "Reference Assemblies" on Windows.
For other F# compilation environments like Xamarin, Mono, Cloud Sharper, FSharp.Formatting, Azure Notebooks, Azure Functions
there was a bit of a mess (made considerably worse by the separation of sigdata/optdata files, [now addressed](https://github.com/Microsoft/visualfsharp/pull/2884))

The F# Core Engineering Group created a NuGet package called [FSharp.Core](https://www.nuget.org/packages/FSharp.Core), partly to avoid
a proliferation of "homebrew" packges appearing at that time. This packge has now grown to be a "one-stop-shop"
package for FSharp.Core for all different target platforms. A single, unified FSharp.Core NuGet packge has advantages:

* Simplicity: Users don't have to think at all - they just reference the package and that's that

* Multi-targeting: One packge supports building against multiple targets

The F# Core Engineering group also publish [notes and guidance on FSharp.Core.dll](http://fsharp.github.io/2015/04/18/fsharp-core-notes.html).

Equally, the FSharp.Core NuGet pacakge has problems, see below.

### Today

As of 2017, ``FSharp.Core.dll`` is very stable in design and the ``FSharp.Core`` NuGet package is in good shape for the main scenarios envisaged by the F# Core Engineering Group. It has prevented
F# users posting new, random packagings of FSharp.Core, and become a trusted part of the F# library ecosystem.  Frequently,
the package reference is managed by [Paket](https://fsprojects.github.io/Paket/), though the package also works well with
NuGet tooling in IDE environments.

As of 2017, Microsoft have a set of [time-critical objectives](https://github.com/Microsoft/visualfsharp/issues/3069) to deliver quality core tooling for F# as
a default part of the dotnet SDK, see [this RFC](https://github.com/fsharp/fslang-design/blob/master/RFCs/FS-1032-fsharp-in-dotnet-sdk.md)
for discussion and pros/cons.

Up to this point, Microsoft have found it difficult to commit to a dependency on the community-provided
FSharp.Core package in the dotnet SDK tooling for a number of reasons

* The package contains some delay-signed DLLs (e.g. the Xamarin variations are delay-signed)

* The package is now relatively large (33MB unzipped, 8MB zipped) and may get bigger (e.g. embedded PDBs).

* The package has been prepared and pushed in an adhoc way

* The package is not pre-installed with tooling (preventing some offline development scenarios)

* The package is not easily buildable from a source-tarball (a legal requirement for Microsoft in some commercial settings)

Some of these problems are easily solvable, others are more work. As a result, early
versions of F# tooling for .NET Core 1.x have used a smaller, signed package
called [Microsoft.FSharp.Core.netcore](https://www.nuget.org/packages/Microsoft.FSharp.Core.netcore/). This contains
nothing but the .NET Standard 1.6 build of FSharp.Core and is small. As a first small step, Microsoft have
agreed to rename this pack ``FSharp.Core.netstandard`` (or possibly ``FSharp.Core.netstandard1.6``)

This means that prior to this PR the FSharp.Core NuGet package is not **yet** in sufficiently good shape to become a
default assumption for the F# support embedded in the .NET SDK. That **doesn't** mean that
the problems aren't solvable. There is long-term value in a unified, simple FSharp.Core package,
and we can **always** iterate towards a better solution. That's just what we need to do.

### Assumptions 

For the purposes of this RFC we will assume

* Portable profiles will eventually be legacy in favour of .NET Standard.  (People building PCL DLLs will still be able to reference an earlier FSharp.Core package)

* Xamarin programmability will eventually iterate towards .NET Standard, at least for the purposes of FSharp.Core. (People building Xamarin apps today will still be able to reference an earlier FSharp.Core package)

* The runtime dependencies in the F# library ecosystem are based around DLL identity (not package identity).

* The compile-time and script-execution-time dependencies in the F# library ecosystem are based on package identity.  

FSharp.Core is a "root dependency" in the F# library ecosystem, both as a DLL with a version and strong name dependency,
and, increasingly, as a package.  Currently, F# libraries tend to assume either

* The Profile 259 version of FSharp.Core, or

* The fatter .NET Framework 4.x version of FSharp.Core

as their "root dependency".  the general principle that **libraries should have a "most portable sufficient" root dependency**,
it makes sense to move to a .NET Standard dependency.

It is **very important** to note that NuGet packages can easily be progressed from a current state to a different state
without breaking existing consumers of specific versions:

* a future version of a package can become "empty" and refer to a different package as an identity, effectively renaming the package (without breaking consumers of existing versions)

* a future version of a package can emit a warning (without breaking consumers of existing versions)

* a future version of a package can **drop** or **add** platforms and dependencies (without breaking consumers of existing versions)

* a future version of a package can be signed by a different authority (without breaking consumers of existing versions)

Together this means that, no matter what the situation at any particular point in time, we can **always** iterate
towards a better world.


### Shared long term goals

All core engineering participants share some common long term goals

* a simple experience for F# tooling users

* a simpler set of FSharp.Core DLLs centered around a .NET Standard version of the library

* the availability of a unified FSharp.Core package

* ongoing binary compatibility for all existing users

* a unified, sensible, healthy, "non-bifurcated" F# library ecosystem

* mutual cooperation to see F# toioling succeed in many different scenarios

* a healthy ecosystem of "innovative" F# tooling

## Scenarios

Please contribute scenarios to the list below.

**Using FSharp.Core NuGet package to simplify .NET Framework development**.  The F# community regularly use the existing FSharp.Core NuGet package to simplify and remove edge cases in .NET Framework development.

1. A C# programmer wants to  consume an F# library (which is assumed not to package FSharp.Core).  Adding a reference to the package doesn't add an FSharp.Core reference to the C# project.  THe F# programmer solves this situation by adding an FSharp.Core NuGet reference to their project, and republishing.

1. Likewise, a C# programmer wants to  consume multiple F# libraries where FSharp.Core dependencies need to be resolved by Paket or NuGet.



## Proposal A

The following steps are proposed for the next few months, until about September 2017:

1. The FSharp.Core package continue to be fully available and usable

2. Microsoft publish ``FSharp.Core.netstandard`` (deprecating ``Microsoft.FSharp.Core.netcore``).  If time permits, Microsoft also publish ``FSharp.Core.netfx`` containing the .NET 4.x DLLs.

3. If technically feasible (see below), F# Core Engineering add these packages as dependencies of a future version of ``FSharp.Core`` and drop the direct inclusion of DLLs. (As noted above these are not necesssarily **permanent** dependencies)

4. ``FSharp.Core.netstandard`` is pre-loaded as a part of dotnet SDK tooling, making some degree of offline development possible.

5. The default build logic for the intial set of F# project templates in the dotnet SDK will be to have a dependency only the ``FSharp.Core.netstandard``
   package.  It will, however, be possible to add ``FSharp.Core`` or other packages as a dependency instead.

Library authors will have a choice of depending on ``FSharp.Core``(unified, fat), ``FSharp.Core.netstandard`` (minimal, needs some thought) or  ``FSharp.Core.netfx`` (not quite minimal, needs some thought) as their FSharp.Core package reference.

.NET Framework Library authors may also simply have no package reference, as is common today.

Looking beyond ~September 2017, we propose:

1. Iterate to improve the FSharp.Core package and ensure its continued usability.

2. Assess the viability of making the .NET Standard version of FSharp.Core be the "basic assumed library" for the F# library ecosystem.

## Proposal B

A second proposal has been developed.

1. Microsoft make and publish the FSharp.Core NuGet package containing **only** the .NET Standard 1.x (likely 1.6) and .NET Framework 4.5 versions of the DLL. Assume this next version is FSharp.Core NuGet package 4.2.0.

Technically, this would be implemented by renaming the authoring Microsoft.FSharp.Core.netcore to FSharp.Core in the http://github.com/Microsoft/visualfsharp repo and adding the ``lib/net40/FSharp.Core.*`` files.

This approach _appears_ to meet 

* Community scenario requirements

* Microsoft scenario requirements

* Microsoft size requrements

* Microsoft publication criteria

* Community simplicity requirements (for a single, unified FSharp.Core package with no additional dependencies).

However, Propoosal B drops support for some existing usage scenarios for FSharp.Core, notably PCL library development.  Those developing PCL libraries would be instructed to continue to use FSharp.Core NuGet package 4.1.17 or before. Tooling such as Paket would automatically ensure that an upgrade didn't happen for people developing PCL libraries.   .NET Standard, .NET Core or .NET Framework consumers of PCL  libraries can of course use FSharp.Core 4.2.0.

Given the guidance that [library authors should target lower versions of FSharp.Core](https://fsharp.github.io/2015/04/18/fsharp-core-notes.html#libraries-target-lower-versions-of-fsharpcore) in order to make their library more useful in more scenarios this seems like a reasonable tradeoff.  Effectively it would be saying __PCL library development is fine until .NET Standard library development is fully supported by all tooling.  However  please stick to referencing NuGet package FSharp.Core 4.1.17 or before, and by the way you will get slightly greater reach for your library if you use NuGet package 4.0.0.1 anyway__. 


###  Problems (Proposal A)

F# Core Engineering previously tried to make ``FSharp.Core`` be a unifying package by using a dependency on ``FSharp.Core.netstandard`` (then called ``Microsoft.FSharp.Core.netcore``).
For some reason that approach failed technically. It is a priority to determine what, if anything, goes wrong with doing that.

Specifically, [this comment](https://github.com/fsharp/fslang-design/issues/188#issuecomment-301245317) indicates that tooling can incorrectly interpret dependencies "FSHarp.Core --> FSharp.Core.netstandard" in an
incorrect way

> It is techinically very very hard to use different packages of FSharp.Core.dll for the same target framework. Is
> a special case not handled, and result in messed transitive dependencies and multiple FSharp.Core.dll referenced.

We should determine the exact nature of this problem and solve it. If it is not solvable in a reasonable time, we will
need to assess the implications of that.


### Open Questions (General)

There are a number of open questions about the way FSharp.Core is referenced by dotnet SDK templates and tooling. These
are somewhat orthogonal to the package structure and delivery. See [this comment](https://github.com/fsharp/fslang-design/issues/188#issuecomment-301245317).

* Is the FSharp.Core package reference pinned or not in templates?  
  - PackageReference support version ranges. Package versions in NuGet are major.minor.patch (semver), that's the official versioning scheme.
  - Pinning to an exact specific version, es 4.1.9 mean if i need to update it, user need to do it manually. And is bad for resilience to bugs. And is more complicated if implicit.
  - Pinning to a wilcard .path (eg ``4.1.*``) mean it's possibile to update it later.
  - Pinning to wilcard minor (eg ``4.*``) mean a more strict contract for the package.
  - Implicit version may help just give the minimal supported version.
  - All bundles (VS/Mono/cli) support offline packages (to not downlaod additional stuff). So this doesnt preclude open ranges, just mean new version need to be downloaded if needed.

* Is the FSharp.Core reference explicit or implicit in .NET SDK project files?  See [again this comment](https://github.com/fsharp/fslang-design/issues/188#issuecomment-301245317)
  - an Fsharp.Core.dll is always required. Template can implicit reference it or not.
  - If implicit, it must be possible to disable it
  - Implicit is one line less in template.
  - Other package managers (like Paket) can just add that property as default, to manage FSharp.Core himself
  - Explicit is easier to understand, and less surprises changing sdk version

### Open Questions (Proposal A)

* Once  ``FSharp.Core.netfx`` is available, will Mono, Xamarin and Visual Studio templates reference this?

* At what point does the ``FSharp.Core`` package drop the inclusion of PCL versions of FSharp.Core (as mentioned above, PCL library development will still always be available by referencing older versions of the package)
  - Answer: when support for .NET Standard package references is widespread, stable and fully accepted

* At what point does the ``FSharp.Core`` package drop the inclusion of Xamarin-specific versions of FSHarp.Core (as mentioned above, Xamaring library development will still always be possible by referencing older versions of the package)
  - Answer: when Xamarin no longer needs these

### Open Questions (Proposal B)

* Is Xamarin development substantially affected?


### Example F# Libraries

Here are a list of some sample F# libraries with .NET Standard or .NET Core compilations and an existing compile-time  dependency on the FSharp.Core NuGet package

* [Expecto](https://github.com/haf/expecto)
* [Logary](https://github.com/logary/logary)
* [Suave](https://github.com/SuaveIO/suave)
* [FSharp.Compiler.Service](https://github.com/fsharp/FSharp.Compiler.Service)
* [Fable compiler](https://github.com/fable-compiler/Fable)
* [FsCheck](https://github.com/fscheck/FsCheck) currently supports .NET, 3 PCL profiles, and .net standard

There are many others - searching github for ``nuget FSharp.Core`` in ``paket.dependencies`` is one way to find them. These libraries can be used to assess the suitabiity of proposals here. For example, does changing these libraries to an FSharp.Core.netstandard dependency birfurcate the world of F# libraries, or can a "FSharp.Core --> FSharp.Core.netstandard" dependency work to make the dependency chain commute.


## Alternatives
[alternatives]: #alternatives

* The dotnet SDK takes a dependency on the existing versions of the FSharp.Core package.
  - Response: See reasons above for issues and what needs to be solved.

* The F# world give up on a unified FSharp.Core package.
  - Response:  No! See reasons above for  why this is not sensible - there are just too many scenarios where is it is just "way simpler and much easier" to instruct
    users to reference this package.

## Acknowledgements

Thanks to Enrico Sada, Steffen Forkmann, Kevin Ransom, Phillip Carter for discussions leading up to the first draft of this RFC.

