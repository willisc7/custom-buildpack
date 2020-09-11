# Custom Buildpack

#### Assumptions
* TBS 1.0.2 is deployed 
    * Note: all steps leading up to the first step that uses the `kp` command can be done without TBS or kpack installed on a kubernetes cluster
* `kp clusterstack list` shows all stacks as **READY**

#### Steps
0. Export git repo dir to an env var to make future commands easier `export ROOT_DIR=$PWD`
0. Install `pack` CLI (https://github.com/buildpacks/pack)
0. `git clone https://github.com/cloudfoundry-samples/spring-music`
0. Set the default builder to the same one TBS uses `pack set-default-builder <CONTAINER_REGISTRY_FQDN>/<TBS_PROJECT>/build-service/default`
    * If you dont have TBS installed, you can just use the Paketo builder be running `pack set-default-builder paketobuildpacks/builder:full`
0. Make sure the image builds with the default builder `pack build spring-music -p $ROOT_DIR/spring-music --trust-builder`
0. When you run `pack build spring-music -p $ROOT_DIR/spring-music -b $ROOT_DIR/sample-buildpack --trust-builder` the following will happen:
    * `sample-buildpack/bin/detect` will be run from the root of the git repo specified (e.g. the `spring-music` directory) and determine whether or not to run whatever is in `sample-buildpack/bin/build`. In this case it will run because `manifest.yml`, inside the `spring-music` directory, contains the string "spring-music"
    * `sample-buildpack/bin/detect` will persist an environment variable called `DEPENDENCIES_DIR` that points to the directory (layer) where it will copy a file called `helloworld`
0. After running the above command, enter the container by running `docker run -it --entrypoint launcher spring-music bash`
0. Navigate to the layer directory that we added using the buildpack `cd $DEPENDENCIES_DIR` and inspect the file we copied in `cat helloworld`
0. Package the buildpack using `pack package-buildpack helloworld-buildpack --config ./package.toml` and see that it now exists in `docker images`
0. Use the packaged buildpack by running `pack build spring-music -p $ROOT_DIR/spring-music -b helloworld-buildpack --trust-builder`
0. Push the buildpack to your container repo `docker tag helloworld-buildpack <CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/<TBS_PROJECT>/helloworld-buildpack` and `docker push <CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/<TBS_PROJECT>/helloworld-buildpack`
0. Add the custom buildpack to the default clusterstore `kp clusterstore add default -b <CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/<TBS_PROJECT>/helloworld-buildpack` and notice that it uploads it as `<CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/build-service/<BUILDPACK_ID>`
0. Edit the default clusterbuilder and add this buildpack to the order
    * `kubectl edit clusterbuilder default`
    * Under `spec.order` add the following lines
        ```
        - group:
          - id: sample-buildpack
        ```
    * Ensure the buildpack is shown in the output of `kp clusterbuilder status default`
0. Build the image using TBS `kp image create spring-music --tag <CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/spring-music --git https://github.com/cloudfoundry-samples/spring-music --git-revision master`
0. Pull the TBS-built container image down and verify the helloworld file is there:
    * `docker pull <CONTAINER_REGISTRY_FQDN>/<SOME_PROJECT>/spring-music`
    * `docker run -it --entrypoint launcher spring-music bash`
    * `cd $DEPENDENCIES_DIR`
    * `cat helloworld`