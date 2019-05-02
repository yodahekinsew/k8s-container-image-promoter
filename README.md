# Container Image Promoter

The Container Image Promoter (aka "cip") promotes images from one Docker
Registry (src registry) to another (dest registry), by reading a Manifest file
(in YAML). The Manifest lists Docker images, and all such images are considered
"blessed" and will be copied from src to dest.

Example Manifest:

```
registries:
- name: gcr.io/myproject-staging-area # publicly readable, does not need a service account for access
  src: true # mark it as the source registry (required)
- name: gcr.io/myproject-production
  service-account: foo@google-containers.iam.gserviceaccount.com
images:
- name: apple
  dmap:
    "sha256:e8ca4f9ff069d6a35f444832097e6650f6594b3ec0de129109d53a1b760884e9": ["1.1", "latest"]
- name: banana
  dmap:
    "sha256:c3d310f4741b3642497da8826e0986db5e02afc9777a2b8e668c8e41034128c1": ["1.0"]
- name: cherry
  dmap:
    "sha256:ec22e8de4b8d40252518147adfb76877cb5e1fa10293e52db26a9623c6a4e92b": ["1.0"]
    "sha256:06fdf10aae2eeeac5a82c213e4693f82ab05b3b09b820fce95a7cac0bbdad534": ["1.2", "latest"]
```

Here, the Manifest cares about 3 images --- `apple`, `banana`, and `cherry`. The
`registries` field lists all destination registries and also the source registry
where the images should be promoted from. To earmark the source registry, it is
called out on its own under the `src-registry` field. In the Example, the
promoter will scan `gcr.io/myproject-staging-area` (*src-registry*) and promote
the images found under `images` to `gcr.io/myproject-production`.

The `src-registry` (staging registry) will always be read-only for the promoter.
Because of this, it's OK to not provide a `service-account` field for it in
`registries`. But in the event that you are trying to promote from one private
registry to another, you would still provide a `service-account` for the staging
registry.

Currently only Google Container Registry (GCR) is supported.

## Renames

It is possible to rename images during the process of promotion. For example, if
you want `gcr.io/myproject-staging-area/apple` to get promoted as
`gcr.io/myproject-production/some/other/subdir/apple`, you could add the
following to the Example above:

```
renames:
- ["gcr.io/myproject-staging-area/apple", "gcr.io/myproject-production/some/other/subdir/apple"]
```

Each entry in the `renames` field is a list of image paths; all images in the
list are treated as "equal". The only requirement is that each list must contain
at least 1 item that points to a source registry (in this case,
`gcr.io/myproject-staging-area`).

# Install

1. Install [bazel][bazel].
2. Run the steps below:

```
go get sigs.k8s.io/k8s-container-image-promoter
cd $GOPATH/src/sigs.k8s.io/k8s-container-image-promoter
make build
```

# Running the Promoter

The promoter relies on calls to `gcloud container images ...` to realize the
intent of the Manifest. It also tries to run the command as the account in
`service-account`. The credentials for this service account must already be set
up in the environment prior to running the promoter.

Given the Example Manifest as above, you can run the promoter with:

```
bazel run -- cip -h -verbosity=3 -manifest=path/to/manifest.yaml
```

Alternatively, you can run the binary directly by examining the bazel output
from running `make build`, and then invoking it with the correct path under
`./bazel-bin`. For example, if you are on a Linux machine, running `make build`
will output a binary at `./bazel-bin/linux_amd64_stripped/cip`.

[bazel]:https://bazel.build/
