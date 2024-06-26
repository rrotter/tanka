---
title: Kustomize support
sidebar:
  order: 3
---

[Kustomize](https://kustomize.io) provides a solution for customizing Kubernetes
manifests in YAML.

Even though Grafana Tanka uses the [Jsonnet language](./jsonnet/overview/) for
resource definition, you can still consume kustomizations, as described below.

:::caution
Keep in mind this feature is considered EXPERIMENTAL
:::

## Consuming a Kustomization from Jsonnet

Kustomize support is provided using the
[`github.com/grafana/jsonnet-libs/tanka-util`](https://github.com/grafana/jsonnet-libs/tree/master/tanka-util)
library. Install it with:

```bash
jb install github.com/grafana/jsonnet-libs/tanka-util
```

The following example shows how to extract the individual resources of the
[`flux2/source-controller`](https://github.com/fluxcd/flux2/tree/main/manifests/bases/source-controller)
kustomization:

```jsonnet
local tanka = import 'github.com/grafana/jsonnet-libs/tanka-util/main.libsonnet';
local kustomize = tanka.kustomize.new(std.thisFile);

{
  source_controller: kustomize.build(path='flux2')
}
```

Kustomize takes a kustomization manifest as input. Go on an create this file
`flux2/kustomization.yaml` relative to above jsonnet:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/fluxcd/flux2/archive/v0.4.3.zip//flux2-0.4.3/manifests/bases/source-controller
```

:::caution
You MUST include the `.new(std.thisFile)` part in the import.
This is what tells Tanka where you actually call `kustomize.build()` from, so
it can find your kustomization manifest.
:::

Once invoked, the `$.source_controller` key holds the individual resources of
the kustomization as a regular Jsonnet object that looks roughly like so:

```jsonnet
{
  'custom_resource_definition_buckets.source.toolkit.fluxcd.io': {/* ... */ },
  'custom_resource_definition_gitrepositories.source.toolkit.fluxcd.io': {/* ... */ },
  'custom_resource_definition_helmcharts.source.toolkit.fluxcd.io': {/* ... */ },
  'custom_resource_definition_helmrepositories.source.toolkit.fluxcd.io': {/* ... */ },
  deployment_source_controller: {/* ... */ },
  service_source_controller: {/* ... */ },
}
```

Above can be [manipulated](./tutorial/environments/#patching) in the same way as
any other Jsonnet data.

## Working with Kustomize

Tanka, like Jsonnet, is hermetic. It **always yields the same resources** when
the project is strictly self-contained.

Kustomize however has the ability to pull
[resources](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/resource/)
from different sources at runtime, which violates above requirement. This is
also apparent in the example above.

:::caution
Due to the nature of Kustomize, it is not feasible to ensure
hermetic and reproducible kustomize builds from within Tanka. Beware of that
when using Kustomize.
:::

## Troubleshooting

### Kustomize executable missing

Kustomize support in Tanka requires the `kustomize` binary installed on your
system and available on the `$PATH`. If Kustomize is not installed, you will see
this error message:

```
evaluating jsonnet: RUNTIME ERROR: Expanding Kustomize: exec: "kustomize": executable file not found in $PATH
```

To solve this, you need to
[install Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/).
If you cannot install it system-wide, you can point Tanka at your executable
using [`TANKA_KUSTOMIZE_PATH`](./env-vars/#tanka_kustomize_path)

### opts.calledFrom unset

This occurs, when Tanka was not told where it `kustomize.build()` was invoked
from. This most likely means you didn't call `new(std.thisFile)` when importing `tanka-util`:

```jsonnet
local tanka = import "github.com/grafana/jsonnet-libs/tanka-util/main.libsonnet";
local kustomize = tanka.kustomize.new(std.thisFile);
                                ↑ This is important
```

### Failed to find kustomization

```
Error: unable to find one of 'kustomization.yaml', 'kustomization.yml' or 'Kustomization' in directory '/home/user/stuff/tanka/environments/default/flux2'
```

Tanka failed to locate your kustomization on the filesystem. It looked at the
relative path you provided in `kustomize.build()`, starting from the directory
of the file you called `kustomize.build()` from.

Please check there is actually a valid kustomization at this place.
