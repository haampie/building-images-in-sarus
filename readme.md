# Building images with kaniko in sarus (rootless patch)

To build images inside of [sarus](https://github.com/eth-cscsc/sarus) one needs a few linux capabilities (`CAP_CHOWN`, `CAP_SETUID`, `CAP_SETGID`, `CAP_FOWNER`, `CAP_DAC_OVERRIDE`). In the current configuration of sarus those capabilities are security problems, but it seems one can still allow them as long as `runc` is run rootless. The patch for that lives in [my rootless branch](https://github.com/haampie/sarus/tree/rootless).

1. ```
   $ git clone -b rootless git@github.com:haampie/sarus.git
   $ cd sarus
   ```
2. Copy the values from `/etc/subuid` and `/etc/subgid` into the `uidMappings` and `gidMappings` in `src/runtime/OCIBundleConfig.cpp` (it's currently hard-coded).
3. Create a spack repo:
   ```
   # Setup Spack bash integration (if you haven't already done so)
   . ${SPACK_ROOT}/share/spack/setup-env.sh

   # Create a local Spack repository for Sarus-specific dependencies
   export SPACK_LOCAL_REPO=${SPACK_ROOT}/var/spack/repos/cscs
   spack repo create ${SPACK_LOCAL_REPO}
   spack repo add ${SPACK_LOCAL_REPO}

   # Import Spack packages for Cpprestsdk, RapidJSON and Sarus
   cp -r <Sarus project root dir>/spack/packages/* ${SPACK_LOCAL_REPO}/packages/
   ```
4. And create a dev-build of sarus:
   ```
   spack dev-build sarus@1.2.0 ~ssh ~configure_installation
   ```
5. It will output the installation path, cd into that (e.g. `cd /home/harmen/spack/opt/spack/linux-ubuntu20.04-zen2/gcc-9.3.0/sarus-1.2.0-u6urc54hwfay77wirfhr4xc4lc56saia`) and run `sudo ./configure_installation.sh`. Next do `sudo vim etc/sarus.json` and set `"securityChecks": false` to make life easier, and set the `rootPath` to some folder writable by the current user.
6. Run `spack load sarus@1.2.0`

7. Pull the kaniko image: `sarus pull gcr.io/kaniko-project/executor`.

8. `cd` into this git repository
9. Copy the `~/.docker/config.json` file, such that `kaniko` can use the credentials to push images to your dockerhub registry.
10. Create a new Dockerfile, or use the one in this repo as an example.
11. Finally run 
    ```
    sarus run --mount=type=bind,src=`pwd`,destination=/workspace --mount=type=bind,src=`pwd`/config.json,destination=/kaniko/.docker/config.json gcr.io/kaniko-project/executor '' --context=/workspace --destination=<username>/my_image --force
    ```
    Note that `--force` is required, because kaniko cannot detect sarus or runc as a runtime, so it will falsely assume it is being run outside of a container (but that's OK).

This should build the image and push it to the registry under `<username>/my_image`.

