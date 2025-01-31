# Resolution

Resolution is the process of taking a list of requirements and converting them to a list of package
versions that fulfill the requirements. Resolution requires recursively searching for compatible
versions of packages, ensuring that the requested requirements are fulfilled and that the
requirements of the requested packages are compatible.

## Dependencies

Most projects and packages have dependencies. Dependencies are other packages that are needed in
order for the current package to work. A package defines its dependencies as _requirements_, roughly
a combination of a package name and acceptable versions. The dependencies defined by the current
project are called _direct dependencies_. The requirements added by each dependency of the current
project are called _indirect_ or _transitive dependencies_.

!!! note

    See the [dependency specifiers
    page](https://packaging.python.org/en/latest/specifications/dependency-specifiers/)
    in the Python Packaging documentation for details about dependencies.

## Basic examples

To help demonstrate the resolution process, consider the following dependencies:

<!-- prettier-ignore -->
- The project depends on `foo` and `bar`.
- `foo` has one version, 1.0.0:
    - `foo 1.0.0` depends on `lib>=1.0.0`.
- `bar` has one version, 1.0.0:
    - `bar 1.0.0` depends on `lib>=2.0.0`.
- `lib` has two versions, 1.0.0 and 2.0.0. Both versions have no dependencies.

In this example, the resolver must find a set of package versions which satisfies the project
requirements. Since there is only one version of both `foo` and `bar`, those will be used. The
resolution must also include the transitive dependencies, so a version of `lib` must be chosen.
`foo 1.0.0` allows all of the available versions of `lib`, but `bar 1.0.0` requires `lib>=2.0.0` so
`lib 2.0.0` must be used.

In some resolutions, there is more than one solution. Consider the following dependencies:

<!-- prettier-ignore -->
- The project depends on `foo` and `bar`.
- `foo` has two versions, 1.0.0 and 2.0.0:
    - `foo 1.0.0` has no dependencies.
    - `foo 2.0.0` depends on `lib==2.0.0`.
- `bar` has two versions, 1.0.0 and 2.0.0:
    - `bar 1.0.0` has no dependencies.
    - `bar 2.0.0` depends on `lib==1.0.0`
- `lib` has two versions, 1.0.0 and 2.0.0. Both versions have no dependencies.

In this example, some version of both `foo` and `bar` must be picked, however, determining which
version requires considering the dependencies of each version of `foo` and `bar`. `foo 2.0.0` and
`bar 2.0.0` cannot be installed together because they conflict on their required version of `lib`,
so the resolver must select either `foo 1.0.0` or `bar 1.0.0`. Both are valid solutions, and
different resolution algorithms may give either result.

## Platform markers

Markers allow attaching an expression to requirements that indicate when the dependency should be
used. For example `bar; python_version<"3.9"` can be used to only require `bar` on Python 3.8 and
older.

Markers are used to adjust a package's dependencies depending on the current environment or
platform. For example, markers can be used to change dependencies based on the operating system, the
CPU architecture, the Python version, the Python implementation, and more.

!!! note

    See the [environment
    markers](https://packaging.python.org/en/latest/specifications/dependency-specifiers/#environment-markers)
    section in the Python Packaging documentation for more details about markers.

Markers are important for resolution because their values change the required dependencies.
Typically, Python package resolvers use the markers of the _current_ platform to determine which
dependencies to use since the package is often being _installed_ on the current platform. However,
for _locking_ dependencies this is problematic — the lockfile would only work for developers using
the same platform the lockfile was created on. To solve this problem, platform-independent, or
"universal" resolvers exist.

uv supports both [platform-specific](#platform-specific-resolution) and
[universal](#universal-resolution) resolution.

## Universal resolution

uv's lockfile (`uv.lock`) is created with a universal resolution and is portable across platforms.
This ensures that dependencies are locked for everyone working on the project, regardless of
operating system, architecture, and Python version. The uv lockfile is created and modified by
[project](../concepts/projects.md) commands such as `uv lock`, `uv sync`, and `uv add`.

universal resolution is also available in uv's pip interface, i.e.,
[`uv pip compile`](../pip/compile.md), with the `--universal` flag. The resulting requirements file
will contain markers to indicate which platform each dependency is relevant for.

During universal resolution, a package may be listed multiple times with different versions or URLs
if different versions are needed for different platforms — the markers determine which version will
be used. A universal resolution is often more constrained than a platform-specific resolution, since
we need to take the requirements for all markers into account.

During universal resolution, a minimum Python version must be specified. Project commands read the
minimum required version from `project.requires-python` in the `pyproject.toml`. When using the pip
interface, provide a value with the `--python-version` option, otherwise the current Python version
will be treated as a lower bound. For example, `--universal --python-version 3.9` writes a universal
resolution for Python 3.9 and later.

Setting the minimum Python version is important because all package versions we select have to be
compatible with the Python version range. For example, a universal resolution of `numpy<2` with
`--python-version 3.8` resolves to `numpy==1.24.4`, while `--python-version 3.9` resolves to
`numpy==1.26.4`, as `numpy` releases after 1.26.4 require at Python 3.9+. Note that we only consider
the lower bound of any Python requirement, upper bounds are always ignored.

## Platform-specific resolution

By default, uv's pip interface, i.e., [`uv pip compile`](../pip/compile.md), produces a resolution
that is platform-specific, like `pip-tools`. There is no way to use platform-specific resolution in
the uv's project interface.

uv also supports resolving for specific, alternate platforms and Python versions with the
`--python-platform` and `--python-version` options. For example, if using Python 3.12 on macOS,
`uv pip compile --python-platform linux --python-version 3.10 requirements.in` can be used to
produce a resolution for Python 3.10 on Linux instead. Unlike universal resolution, during
platform-specific resolution, the provided `--python-version` is the exact python version to use,
not a lower bound.

!!! note

    Python's environment markers expose far more information about the current machine
    than can be expressed by a simple `--python-platform` argument. For example, the `platform_version` marker
    on macOS includes the time at which the kernel was built, which can (in theory) be encoded in
    package requirements. uv's resolver makes a best-effort attempt to generate a resolution that is
    compatible with any machine running on the target `--python-platform`, which should be sufficient for
    most use cases, but may lose fidelity for complex package and platform combinations.

## Dependency preferences

If resolution output file exists, i.e. a uv lockfile (`uv.lock`) or a requirements output file
(`requirements.txt`), uv will _prefer_ the dependency versions listed there. Similarly, if
installing a package into a virtual environment, uv will prefer the already installed version if
present. This means that locked or installed versions will not change unless an incompatible version
is requested or an upgrade is explicitly requested with `--upgrade`.

## Resolution strategy

By default, uv tries to use the latest version of each package. For example,
`uv pip install flask>=2.0.0` will install the latest version of Flask, e.g., 3.0.0. If
`flask>=2.0.0` is a dependency of the project, only `flask` 3.0.0 will be used. This is important,
for example, because running tests will not check that the project is actually compatible with its
stated lower bound of `flask` 2.0.0.

With `--resolution lowest`, uv will install the lowest possible version for all dependencies, both
direct and indirect (transitive). Alternatively, `--resolution lowest-direct` will use the lowest
compatible versions for all direct dependencies, while using the latest compatible versions for all
other dependencies. uv will always use the latest versions for build dependencies.

For example, given the following `requirements.in` file:

```python title="requirements.in"
flask>=2.0.0
```

Running `uv pip compile requirements.in` would produce the following `requirements.txt` file:

```python title="requirements.txt"
# This file was autogenerated by uv via the following command:
#    uv pip compile requirements.in
blinker==1.7.0
    # via flask
click==8.1.7
    # via flask
flask==3.0.0
itsdangerous==2.1.2
    # via flask
jinja2==3.1.2
    # via flask
markupsafe==2.1.3
    # via
    #   jinja2
    #   werkzeug
werkzeug==3.0.1
    # via flask
```

However, `uv pip compile --resolution lowest requirements.in` would instead produce:

```python title="requirements.in"
# This file was autogenerated by uv via the following command:
#    uv pip compile requirements.in --resolution lowest
click==7.1.2
    # via flask
flask==2.0.0
itsdangerous==2.0.0
    # via flask
jinja2==3.0.0
    # via flask
markupsafe==2.0.0
    # via jinja2
werkzeug==2.0.0
    # via flask
```

When publishing libraries, it is recommended to separately run tests with `--resolution lowest` or
`--resolution lowest-direct` in continuous integration to ensure compatibility with the declared
lower bounds.

## Pre-release handling

By default, uv will accept pre-release versions during dependency resolution in two cases:

1. If the package is a direct dependency, and its version specifiers include a pre-release specifier
   (e.g., `flask>=2.0.0rc1`).
1. If _all_ published versions of a package are pre-releases.

If dependency resolution fails due to a transitive pre-release, uv will prompt use of
`--prerelease allow` to allow pre-releases for all dependencies.

Alternatively, the transitive dependency can be added as a [constraint](#dependency-constraints) or
direct dependency (i.e. in `requirements.in` or `pyproject.toml`) with a pre-release version
specifier (e.g., `flask>=2.0.0rc1`) to opt-in to pre-release support for that specific dependency.

Pre-releases are
[notoriously difficult](https://pubgrub-rs-guide.netlify.app/limitations/prerelease_versions) to
model, and are a frequent source of bugs in other packaging tools. uv's pre-release handling is
_intentionally_ limited and requires user opt-in for pre-releases to ensure correctness.

For more details, see
[Pre-release compatibility](../pip/compatibility.md#pre-release-compatibility).

## Dependency constraints

uv supports constraints files (`--constraint constraints.txt`), like pip, which narrow the set of
acceptable versions for the given packages. Constraint files are like a regular requirements files,
but they do not add packages to the requirements — they only take affect if the package is requested
in a direct or transitive dependency. Constraints are often useful for reducing the range of
available versions for a transitive dependency without adding a direct requirement on the package.

## Dependency overrides

Overrides allow bypassing failing or undesirable resolutions by overriding the declared dependencies
of a package. Overrides are a useful last resort for cases in which the you know that a dependency
is compatible with a newer version of a package than it declares, but the it has not yet been
updated to declare that compatibility.

For example, if a transitive dependency declares the requirement `pydantic>=1.0,<2.0`, but _works_
with `pydantic>=2.0`, the user can override the declared dependency with `pydantic>=1.0,<3` to allow
the resolver to installer a newer version of `pydantic`.

While constraints and dependencies are purely additive, and thus cannot expand the set of acceptable
versions for a package, overrides can expand the set of acceptable versions for a package, providing
an escape hatch for erroneous upper version bounds. As with constraints, overrides do not add a
dependency on the package and only take affect if the package is requested in a direct or transitive
dependency.

In a `pyproject.toml`, use `tool.uv.override-dependencies` to define a list of overrides. In the
pip-compatible interface, the `--override` option can be used to pass files with the same format as
constraints files.

If multiple overrides are provided for the same package, they must be differentiated with
[markers](#platform-markers). If a package has a dependency with a marker, it is replaced
unconditionally when using overrides — it does not matter if the marker evaluates to true or false.

## Lower bounds

By default, `uv add` adds lower bounds to dependencies and, when using uv to manage projects, uv
will warn if direct dependencies don't have lower bound.

Lower bounds are not critical in the "happy path", but they are important for cases where there are
dependency conflicts. For example, consider a project that requires two packages and those packages
have conflicting dependencies. The resolver needs to check all combinations of all versions within
the constraints for the two packages — if all of them conflict, an error is reported because the
dependencies are not satisfiable. If there are no lower bounds, the resolver can (and often will)
backtrack down to the oldest version of a package. This isn't only problematic because it's slow,
the old version of the package often fails to build, or the resolver can end up picking a version
that's old enough that it doesn't depend on the conflicting package, but also doesn't work with your
code.

Lower bounds are particularly critical when writing a library. It's important to declare the lowest
version for each dependency that your library works with, and to validate that the bounds are
correct — testing with [`--resolution lowest` or `resolution lowest-direct`](#resolution-strategy).
Otherwise, a user may receive an old, incompatible version of one of your library's dependencies and
the library will fail with an unexpected error.

## Reproducible resolutions

uv supports an `--exclude-newer` option to limit resolution to distributions published before a
specific date, allowing reproduction of installations regardless of new package releases. The date
may be specified as an [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339.html) timestamp (e.g.,
`2006-12-02T02:07:43Z`) or a local date in the same format (e.g., `2006-12-02`) in your system's
configured time zone.

Note the package index must support the `upload-time` field as specified in
[`PEP 700`](https://peps.python.org/pep-0700/). If the field is not present for a given
distribution, the distribution will be treated as unavailable. PyPI provides `upload-time` for all
packages.

To ensure reproducibility, messages for unsatisfiable resolutions will not mention that
distributions were excluded due to the `--exclude-newer` flag — newer distributions will be treated
as if they do not exist.

## Source distribution

Most packages publish their source distributions as gzip tarball (`.tar.gz`) or zip (`.zip`)
archives. While less common, other formats are also supported when reading and extracting source
distributions:

- bzip2 tarball (`.tar.bz2`)
- gzip tarball (`.tar.gz`)
- xz tarball (`.tar.xz`)
- zip (`.zip`)
- zstd tarball (`.tar.zst`)

## Learn more

For more details about the internals of the resolver, see the
[resolver reference](../reference/resolver-internals.md) documentation.
