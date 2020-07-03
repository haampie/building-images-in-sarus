# Building images with kaniko in sarus

1. It seems the minimal set of linux capabilities necessary to build a typical docker image is `CAP_CHOWN`, `CAP_SETUID`, `CAP_SETGID`, `CAP_FOWNER`, `CAP_DAC_OVERRIDE`. By default [sarus](https://github.com/eth-cscs/sarus) has none of these enabled, so that requires a [minimal change](https://github.com/haampie/sarus/commit/52616636f270f041ac8b13f4b7e596d5bc9d5ad3) to the code base. 
   ```
   $ git clone git@github.com:haampie/sarus.git
   $ cd sarus
   ```
2. Further on my pc I had [some issues](https://github.com/eth-cscs/sarus/issues/14) with the number of cpus requested by sarus. In case `cat /proc/self/status | grep Cpus_allowed_list` does not match `nproc`, it can be useful to hard-code the number of requested cpus to something that corresponds with `nproc` as done here: https://github.com/haampie/sarus/commit/76570aa8a283f5df1df3c3dcdfae116460a803f6.
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
5. It will output the installation path, cd into that (e.g. `cd /home/harmen/spack/opt/spack/linux-ubuntu20.04-zen2/gcc-9.3.0/sarus-1.2.0-u6urc54hwfay77wirfhr4xc4lc56saia`) and run `sudo ./configure_installation.sh`. Next do `sudo vim etc/sarus.json` and set `"securityChecks": false` to make life easier.
6. Run `spack load sarus@1.2.0` and test whether `sarus` works

7. Now in my case I pulled the `kaniko` executor image from google and pushed it to my own dockerhub account/registry, because sarus did not automatically pull from gcr.io: `docker pull gcr.io/kaniko-project/executor`, `docker tag gcr.io/kaniko-project/executor <username>/executor`, `docker push <username>/executor`.

8. Next, pull the image with sarus: `sarus pull <username>/executor`.

9. `cd` into this git repository again.
10. Copy the `~/.docker/config.json` file, such that `kaniko` can use the credentials to push images to your dockerhub registry.
11. Create a new Dockerfile, or use the one in this repo as an example.
12. In my case I had to work around quirks of kaniko, where it had an empty `/etc/resolv.conf` file when the image is being built. So on your host machine, run `cat /etc/resolve.conf` and copy the nameserver bit. In your Dockerfile add a `RUN echo "nameserver ..." > /etc/resolv.conf` statement to make sure DNS will work.
13. Make sure to add a `USER root` statement such that things like `apt-get install` work.
14. Make sure you have the `Dockerfile` and `config.json` in your current working directory
15. Finally run 
    ```
    sarus run --mount=type=bind,src=`pwd`,destination=/workspace --mount=type=bind,src=`pwd`/config.json,destination=/kaniko/.docker/config.json <username>/executor '' --context=/workspace --destination=<username>/my_image --force
    ```
    Note that `--force` is required, because kaniko cannot detect sarus or runc as a runtime, so it will falsely assume it is being run outside of a container (but that's OK).

This should build the image and push it to the registry under `<username>/my_image`.

