# Docker Build File Examples

Just short examples and explantion to overriding ENTRYPOINTS and also using DOCKER_BUILDKIT to speed up execution

## Overridding ENTRYPOINT and CMD

Just a test of stages and overridding ENTRYPOINTS between stages/files

### Separate Dockerfiles

Running build to create the docker images
```cmd
# Must be run before the build files as this is used as a base
docker build -f ./Dockerfile.first -t override-entrypoint .

docker build -f ./Dockerfile.second -t override-entrypoint-2 .
docker build -f ./Dockerfile.third -t override-entrypoint-3 .
docker build -f ./Dockerfile.fourth -t override-entrypoint-4 .
```

Running the images to create the containers
```cmd
docker run override-entrypoint:latest
# -> "Helloworld"

docker run override-entrypoint-2:latest
# -> "Helloworld"

docker run override-entrypoint-3:latest
# -> "Hello overridden"

docker run override-entrypoint-3:latest World
# -> "Hello overridden World"

docker run override-entrypoint-4:latest
# -> "Hello world /bin/echo Hello overridden"

docker run override-entrypoint-4:latest "More stuff"
# -> "Hello world /bin/echo Hello world More Stuff"
```

As can be seen if adding a new entrypoint then the new one is used, otherwise it will use the original. If CMD is used to override it is simply adding to the end, however if stuff is appended to the run command then that will add to ENTRYPOINT but discard the CMD and append the extra.

### Multi Stage Dockerfile

I created a multi stage to test if that made a difference, using the target arg (see below), but the results are the same.

```cmd
docker build -f ./Dockerfile.multi --target=third -t override-entrypoint-multi .
```
```
docker run override-entrypoint-multi:latest
```

## Unnecessary Build Steps

Currently docker default build behaviour is to run all the stages in order until it hits the last stage, or hits the `--target` stage. However when looking at the multistage build file `Dockerfile.multi` we can see that the second/third/forth stages do not depend on each other and this is wasting resources (important if you are paying for build minutes!).

*A note to mention docker attempts to cache build stages anyway, so this is not the deal breaker it may seem, but its not ideal especially if using docker in docker as the caching will not be available between builds.*

### DOCKER_BUILDKIT

Docker has a solution for this, however it is 'opt-in', by setting the environment variable DOCKER_BUILDKIT=1 we tell Docker to use its BuildKit (available since 18.09) which is a more efficient set of build tools and supports only building the stages that are needed for a target.

To build the third stage without the second stage building:
```
DOCKER_BUILDKIT=1 docker build -f ./Dockerfile.multi --target=third -t override-entrypoint-multi .
```
Output:
```
[+] Building 1.7s (7/7) FINISHED
 => [internal] load .dockerignore                                                                                  0.1s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load build definition from Dockerfile.multi                                                         0.2s
 => => transferring dockerfile: 44B                                                                                0.0s
 => [internal] load metadata for docker.io/alpine/git:latest                                                       0.0s
 => CACHED [override-entrypoint 1/2] FROM docker.io/alpine/git                                                     0.0s
 => [override-entrypoint 2/2] RUN echo "First stage output"                                                        0.4s
 => [third 1/1] RUN echo "Third stage output"                                                                      0.7s
 => exporting to image                                                                                             0.4s
 => => exporting layers                                                                                            0.3s
 => => writing image sha256:7e626732810a2139d44df6227191cf0b662a8b9baa31ce0e75d3c1b79b268024                       0.0s
 => => naming to docker.io/library/override-entrypoint-multi
```

You can see that the "First stage output" and "Third stage output" are present but not "Second stage output".

Below is the output without using BuildKit and we can see that its still showing "Second stage output". Additionally the output is more concise and the whole process with BuildKit took 1.7s but the traditionaly build to 5.6s (using the time command to calculate) showing a marked speed improvement.

```
docker build -f ./Dockerfile.multi --target=third -t override-entrypoint-multi .
```
```
Sending build context to Docker daemon  51.71kB
Step 1/8 : FROM alpine/git as override-entrypoint
 ---> 04dbb58d2cea
Step 2/8 : RUN echo "First stage output"
 ---> Running in 779eee94658a
First stage output
Removing intermediate container 779eee94658a
 ---> 5f3d8c01aafa
Step 3/8 : ENTRYPOINT ["/bin/echo", "Hello world"]
 ---> Running in 212aa4ade024
Removing intermediate container 212aa4ade024
 ---> cacaa6328138
Step 4/8 : FROM override-entrypoint as second
 ---> cacaa6328138
Step 5/8 : RUN echo "Second stage output"
 ---> Running in 3ade693ce0c9
Second stage output
Removing intermediate container 3ade693ce0c9
 ---> 421aa157454e
Step 6/8 : FROM override-entrypoint as third
 ---> cacaa6328138
Step 7/8 : RUN echo "Third stage output"
 ---> Running in 31d675c2a775
Third stage output
Removing intermediate container 31d675c2a775
 ---> 3ae3e4a9c393
Step 8/8 : ENTRYPOINT [ "/bin/echo", "Hello overridden" ]
 ---> Running in c8f2d1a800d3
Removing intermediate container c8f2d1a800d3
 ---> 444858e9a1a9
Successfully built 444858e9a1a9
Successfully tagged override-entrypoint-multi:latest
```

## References

- https://goinbigdata.com/docker-run-vs-cmd-vs-entrypoint/
- https://docs.docker.com/develop/develop-images/build_enhancements/#to-enable-buildkit-builds
- https://docs.docker.com/develop/develop-images/build_enhancements/#to-enable-buildkit-builds