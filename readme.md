# Building images with kaniko in sarus (rootless patch)

To build images inside of [sarus](https://github.com/eth-cscsc/sarus) one needs a few linux capabilities (`CAP_CHOWN`, `CAP_SETUID`, `CAP_SETGID`, `CAP_FOWNER`, `CAP_DAC_OVERRIDE`). In the current configuration of sarus those capabilities are security problems, but it seems one can still allow them as long as `runc` is run rootless. The patch for that lives in [my rootless branch](https://github.com/haampie/sarus/tree/rootless).

1. ```
   $ git clone -b rootless git@github.com:haampie/sarus.git
   $ cd sarus
   $ spack repo add spack/
   ```
2. Copy the values from `/etc/subuid` and `/etc/subgid` into the `uidMappings` and `gidMappings` in `src/runtime/OCIBundleConfig.cpp` (it's currently hard-coded).
3. And create a dev-build of sarus:
   ```
   spack dev-build sarus@1.3.1 ~ssh ~configure_installation
   ```
4. It will output the installation path, cd into that (e.g. `cd /home/harmen/spack/opt/spack/linux-ubuntu20.04-zen2/gcc-9.3.0/sarus-1.3.1-u6urc54hwfay77wirfhr4xc4lc56saia`) and run `sudo ./configure_installation.sh`. Next do `sudo vim etc/sarus.json` and set `"securityChecks": false` to make life easier, and set the `rootPath` to some folder writable by the current user.
5. Run `spack load sarus@1.3.1`

6. Pull the kaniko image: `sarus pull gcr.io/kaniko-project/executor`.

7. `cd` into this git repository
8. Copy the `~/.docker/config.json` file, such that `kaniko` can use the credentials to push images to your dockerhub registry.
9. Create a new Dockerfile, or use the one in this repo as an example.
10. Finally run 
    ```
    sarus run --mount=type=bind,src=`pwd`,destination=/workspace --mount=type=bind,src=`pwd`/config.json,destination=/kaniko/.docker/config.json gcr.io/kaniko-project/executor '' --context=/workspace --destination=<username>/my_image --force
    ```
    Note that `--force` is required, because kaniko cannot detect sarus or runc as a runtime, so it will falsely assume it is being run outside of a container (but that's OK).

This should build the image and push it to the registry under `<username>/my_image`.

