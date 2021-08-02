# How I have reduced our nodejs docker image size by 70%?

This article is based on my work for several clients that I've worked for in the past. I would like to share my experience with you. Before the work, the image size is around `1.5GB` and post the below mentioned steps size is reduced to `446MB` which is almost reduction of `70.3%` in size.

## Key things

* [Moving to alpine image](#moving-to-alpine-image)
* [Utilizing docker multi-stage build](#utilizing-docker-multi-stage-build)
* [Copying only the required files to the final image](#copying-only-the-required-files-to-the-final-image)

### Moving to alpine image

The first step I have taken to reduce size is to move our base image from node to node alpine image. In our case, I've used `node:16.4-alpine3.14` image with very little size of `~39MB` instead of `node:16.4` image which is having size of `~330MB`. Here itself we have reduced almost `291MB` of size from our final image.

### Utilizing docker multi-stage build

All dev dependencies can be dropped from our final image as they are only required during build phase. Sometimes we might need to install packages to support building the application. A simple example to illustrate what I am saying is, we required to install `g++`, `make`, `python3` packages so that `node-gyp` can build the native modules. I have used `npm prune --production` to remove all dev dependencies from node modules.

* Running `npm prune --production`, reduced almost `190MB (522MB-332MB)` size of node modules in our case.

### Copying only the required files to the final image

Once the project is built, we can simply copy below mentioned files to the final image. We don't need any other dev dependencies or any other intermediate files from builder stage.

* package.json & package-lock.json
* other application resources if any
* node_modules
* build output directory

## To improve your build time

* Before copying the entire source code, copying `package.json` and `package-lock.json` and then running `npm install` will speed up the subsequent builds.

* Before copying the build output and node modules, we should copy required application resources. So that docker won't rebuild the layer if the file contents doesn't change.

## Full dockerfile at a glance

```dockerfile
FROM node:16.4-alpine3.14 as builder

WORKDIR /app

# g++ make python3 required by node-gyp
RUN apk add --no-cache g++ make python3

# Installs required node packages
COPY package*.json .npmrc /app/
RUN npm install

# Builds node application
COPY . .
RUN npm run build && npm prune --production

# ==== Final Image
FROM node:16.4-alpine3.14 as final
USER node:node
WORKDIR /app

# Copying required resources
COPY --chown=node:node resources resources

# Copying build output
COPY --from=builder --chown=node:node /app/package*.json ./
COPY --from=builder --chown=node:node /app/build/ build

# Copying production only node modules
COPY --from=builder --chown=node:node /app/node_modules/ node_modules

CMD ["node", "build/src/server.js"]

```

## Conclusion and Next Steps

By doing the above simple steps, we have reduced almost our image size by 70%. And I think if you follow the steps above, you can reduce your node image size by even more. Make sure you run all your unit and performance tests before you push your image to production.

* Use [Dive](https://github.com/wagoodman/dive#dive) tool to see where exactly image size can be reduced. This tool has nice UI to visualize files which are created / modified between the layers.

* Try to dig deeper into the application dependencies and see if you can remove unused dependencies.

* See which node modules are having more size using `du` and try to remove the unnecessary ones.

* Always to to use specific package(s) instead of the whole package. Ex: `@aws-sdk/client-s3` instead of `aws-sdk`.
