# hms-runner
a lightweight hms runner in linux for cloud-compute

# building this project
To build this project follow these steps
1. download code locally
2. open project with vs code
3. if you do not have remote development extensions, download those
4. if you do not have docker desktop, download it
5. reopen project inside Dockerfile container in the root of the project
6. inside the container, install java and gradle extensions
7. add ExcessPrecip.p01.hdf and ElkMiddle.p01.hdf in testdata folder
8. run tests.

# Versions supported
The CICD pipeline is currently set up for version 4.13+ of HEC-HMS.
Tags for release `vx.x.x`:

| Tag | Meaning |
| --- | --- |
| `vx.x.x-hms4.13` | immutable - pin this in a compute manifest or plugin definition |
| `hms4.13` | floating, always the newest runner built against HEC-HMS 4.13 |
| `latest` | floating, follows whichever version is marked `default: true` in the matrix |

Supported now: **4.13**. Deliberately not built, and why:

- **4.14** - the artifact downloads fine, but the plugin does not currently
  compile against it. Commented out in the matrix; re-enable after the compile
  errors are fixed.
- **4.12** - builds and starts, but HEC-HMS 4.12 has no excess-precipitation
  export action, so an image made from it cannot serve real workloads.
- **4.13-beta.6** - superseded by the 4.13 release.
- **4.15-beta.2** - HEC publishes no Linux build of it. **4.15-beta.1** does have
  one and can be added to the matrix when wanted.

## Running the test model against a published image

The Muncie (White River) example model is committed under `testdata/muncie`, clone the project locally and use a published image to run the compute locally. From the repository root, in PowerShell:

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

`--env-file` supplies the run configuration, and the two `-v` flags mount the
payload and the model. Substitute the image reference to test a different
published version or HEC-HMS build; the package is public, so no `docker login`
is required.

A successful run ends like this:

```
Picked up JAVA_TOOL_OPTIONS: -Djava.library.path=/opt/hms/bin/gdal:/opt/hms/bin:...
hmsrunner says hello.
opening project /model/muncie/WhiteRiver_Muncie.hms

preparing to run Simulation Dec2013
Simulation run completed for /model/muncie/WhiteRiver_Muncie.hms
```

`Simulation run completed` is the line to check for.
