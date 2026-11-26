# hms-runner

a lightweight hms runner in linux for cloud-compute

# building this project

To build this project follow these steps

1. download code locally
2. open project with vs code
3. if you do not have remote development extensions, download those
4. if you do not have docker desktop, download it
5. reopen project inside Dockerfile container in the root of the project --
   `.devcontainer/devcontainer.json` selects which (`Dockerfile_14` for HEC-HMS 4.14,
   `Dockerfile_13_6` for 4.13)
6. the java and gradle extensions are installed by the dev container
   (`customizations.vscode.extensions`), so this no longer needs doing by hand
7. add ExcessPrecip.p01.hdf and ElkMiddle.p01.hdf in testdata folder
8. run tests.

Inside a dev container `gradle build` needs no `-P` flags: the image exports
`HMS_HOME` and `JAVA_COMPAT`, `build.gradle` defaults `hmsHome` to `/hms` (which is
what `.vscode/launch.json` uses too), and the cc-java-sdk fat jar is resolved from
its GitHub release, so no `deps/` drop is needed.

# Versions supported

A HEC-HMS release pins a whole toolchain, not just an artifact. These have to agree,
or the build fails in a way that looks like an API change and is not one:

| HEC-HMS | hms.jar bytecode | builder JDK | Gradle to run it | runner bytecode | runtime JRE | `libjavaHeclib.so` |
| --- | --- | --- | --- | --- | --- | --- |
| 4.13 | 61.0 (Java 17) | eclipse-temurin:17-jdk | 7.3.1 | 17 | openjdk-17 (ubuntu:20.04) | `bin/` |
| 4.14 | 65.0 (Java 21) | eclipse-temurin:21-jdk | 9.7.1 | 21 | openjdk-21 (ubuntu:24.04) | `bin/javaHeclib/` |

Why each column exists, all of it learned from 4.14:

* **builder JDK** - javac cannot read a class file newer than itself. Java 17 against
  4.14 produced 26 errors beginning
  `class file has wrong version 65.0, should be 61.0`.
* **Gradle version** - Gradle 7.3.1 cannot *execute* on Java 21 (it supports 8-17), so
  the JDK and Gradle move together; Java 21 needs Gradle >= 8.5.
* **runner bytecode** - must not exceed the runtime JRE, or the container starts with
  `UnsupportedClassVersionError`.
* **runtime base image** - must actually ship that JRE: ubuntu:20.04 has no
  `openjdk-21-jre-headless`.
* **`libjavaHeclib.so`** - 4.14 moved it into `bin/javaHeclib/`. Compiling and starting
  both succeed without it; `Project.open` then dies with
  `UnsatisfiedLinkError: no javaHeclib in java.library.path`.

The matrix in `.github/workflows/build-push-ghcr.yml` is the single source of truth:
one row per release carrying `hms`, `builderImage`, `gradleVersion`, `javaCompat`,
`runtimeJava`, `runtimeBaseImage` (plus an optional `tarballUrl`), each passed to
`Dockerfile.prod` as a build-arg. The same values are written into the image's OCI
labels (`io.github.usace-cloud-compute.hms.*`), so a published tag can be asked what
it was built with.

Tags for release `vx.x.x`:

| Tag | Meaning |
| --- | --- |
| `vx.x.x-hms4.13` | immutable - pin this in a compute manifest or plugin definition |
| `hms4.13` | floating, always the newest runner built against HEC-HMS 4.13 |
| `latest` | floating, follows whichever version is marked `default: true` in the matrix |

Supported now: **4.13** and **4.14**.

Deliberately not built, and why:

* **4.12** - builds and starts, but HEC-HMS 4.12 has no excess-precipitation export
  action, so an image made from it cannot serve real workloads.
* **4.13-beta.6** - superseded by the 4.13 release.
* **4.15-beta.2** - HEC publishes no Linux build of it. **4.15-beta.1** does have one;
  add a matrix row for it (expect the Java 21 / Gradle 9 toolchain and `bin/javaHeclib`
  like 4.14) and dry-run it before trusting the answer.

> 4.14 was disabled on 2026-11-26 as "does not currently compile against it". That was
> a misdiagnosis: it never compiled because it needs Java 21 and Gradle 8.5+. Once it
> did, the remaining problem was the relocated heclib native. No HEC-HMS API turned out
> to have changed.

## Running the test model against a published image

The Muncie (White River) example model is committed under `testdata/muncie`, clone the
project locally and use a published image to run the compute locally. From the
repository root, in PowerShell:

```powershell
docker run --env-file .\testdata\muncie\.env-mnt `
  -v .\testdata\muncie\mnt-payload:/model/mnt-payload `
  -v .\testdata\muncie\hms:/mount `
  ghcr.io/usace-cloud-compute/hms-runner:v0.0.6-hms4.13
```

On Linux or macOS the same invocation is:

```sh
docker run --env-file testdata/muncie/.env-mnt \
  -v "$PWD/testdata/muncie/mnt-payload:/model/mnt-payload" \
  -v "$PWD/testdata/muncie/hms:/mount" \
  ghcr.io/usace-cloud-compute/hms-runner:v0.0.6-hms4.13
```

`--env-file` supplies the run configuration, and the two `-v` flags mount the payload
and the model; `/mount` must be writable, because the output is copied back through the
store rooted there. Substitute the image reference to test a different published
version or HEC-HMS build; the package is public, so no `docker login` is required.

A successful run ends like this:

```
Picked up JAVA_TOOL_OPTIONS: -Djava.library.path=/opt/hms/bin/gdal:/opt/hms/bin:...
hmsrunner says hello.
opening project /model/muncie/WhiteRiver_Muncie.hms
preparing to run Simulation Dec2013
Simulation run completed for /model/muncie/WhiteRiver_Muncie.hms
```

`Simulation run completed` is the line to check for -- and check the output file too,
because that line is printed before the result is copied back to the store. Anything
containing `Cannot find proj.db` is a failure even if the run completes: the compute
finished without projection data, so the numbers are suspect. The CI smoke test runs
exactly this model for every leg and asserts both.
