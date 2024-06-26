---
title: Abstraction
sidebar:
  order: 5
---

While we won't need to touch the resource definitions directly that frequently
anymore now that we have the `_config` object for our tunables, the
`main.jsonnet` file is still very long and hard to read. Especially because of
all the brackets, it's even worse than yaml at the moment.

## Splitting it up

Let's start cleaning this up by separating logical pieces into distinct files:

- `main.jsonnet`: Still our main file, containing the `_config` object and importing the other files
- `grafana.jsonnet`: `Deployment` and `Service` for the Grafana instance
- `prometheus.jsonnet`: `Deployment` and `Service` for the Prometheus server

```jsonnet
// /environments/default/grafana.jsonnet
{
  // DO NOT use the root level here.
  // Include the grafana subkey, otherwise $ won't work.
  grafana: {
    deployment: {
      apiVersion: 'apps/v1',
      kind: 'Deployment',
      metadata: {
        name: $._config.grafana.name,
      },
      spec: {
        selector: {
          matchLabels: {
            name: $._config.grafana.name,
          },
        },
        template: {
          metadata: {
            labels: {
              name: $._config.grafana.name,
            },
          },
          spec: {
            containers: [
              {
                image: 'grafana/grafana',
                name: $._config.grafana.name,
                ports: [{
                    containerPort: $._config.grafana.port,
                    name: 'ui',
                }],
              },
            ],
          },
        },
      },
    },
    service: {
      apiVersion: 'v1',
      kind: 'Service',
      metadata: {
        labels: {
          name: $._config.grafana.name,
        },
        name: $._config.grafana.name,
      },
      spec: {
        ports: [{
            name: '%s-ui' % $._config.grafana.name,
            port: $._config.grafana.port,
            targetPort: $._config.grafana.port,
        }],
        selector: {
          name: $._config.grafana.name,
        },
        type: 'NodePort',
      },
    },
  }
}
```

The file should contain just the same that was located under the `grafana` key
on the root object before. Do the same for `/environments/default/prometheus.jsonnet` as well.

```jsonnet
// /environments/default/main.jsonnet
// Think of `import` as copy-pasting the contents
// of ./grafana.jsonnet here
(import "grafana.jsonnet") +
(import "prometheus.jsonnet") +
{
  _config:: {
    grafana: {
      port: 3000,
      name: "grafana",
    },
    prometheus: {
      port: 9090,
      name: "prometheus"
    }
  }
}
```

:::note[Clarification]
It might seem odd at first sight, that this code works, because
`grafana.jsonnet` still refers to the root object using `$`, even
though it is outside of the file's scope.  
However, Jsonnet is lazy-evaluated which means that the contents of
`grafana.jsonnet` are **first "copied"** into `main.jsonnet` (the root
object) and **then evaluated**. This means the above code actually consists of
all three objects joined to one big object, which is then converted to JSON.
:::

## Helper utilities

While `main.jsonnet` is now short and very readable, the other two files are not
really an improvement over regular yaml, mostly because they are still full of
boilerplate.

Let's use functions to create some useful helpers to reduce the amount of
repetition. For that, we create a new file called `kubernetes.libsonnet`, which
will hold our Kubernetes utilities.

:::note
The extension for Jsonnet libraries is `.libsonnet`. While you do
not have to use it, it distinguishes helper code from actual configuration.
:::

### A Deployment constructor

Creating a `Deployment` requires some mandatory information and a lot of
boilerplate. A function that creates one could look like this:

```jsonnet
{
  // hidden k namespace for this library
  k:: {
    deployment: {
      new(name, containers): {
        apiVersion: "apps/v1",
        kind: "Deployment",
        metadata: {
          name: name,
        },
        spec: {
          selector: { matchLabels: {
            name: name,
          }},
          template: {
            metadata: { labels: {
              name: name,
            }},
            spec: { containers: containers }
          }
        }
      }
    }
  }
}
```

Invoking this function will substitute all the variables with the respective
passed function parameters and return the assembled object.

To use it, just add it to the root object in `main.jsonnet`:

```jsonnet
  (import "kubernetes.libsonnet") + // this line adds it
  (import "grafana.jsonnet") +
  (import "prometheus.jsonnet") +
  { /* ... */ }
```

Let's simplify our `grafana.jsonnet` a bit:

```jsonnet
{
  grafana: {
    deployment: $.k.deployment.new("grafana", [{
      image: 'grafana/grafana',
      name: 'grafana',
      ports: [{
          containerPort: 3000,
          name: 'ui',
      }],
    }]),
    service: {
      apiVersion: 'v1',
      kind: 'Service',
      metadata: {
        labels: {
          name: 'grafana',
        },
        name: 'grafana',
      },
      spec: {
        ports: [{
            name: 'grafana-ui',
            port: 3000,
            targetPort: 3000,
        }],
        selector: {
          name: 'grafana',
        },
        type: 'NodePort',
      },
    },
  }
}
```

This drastically simplified the creation of the `Deployment`, because we do not
need to remember how exactly a `Deployment` is structured anymore. Just call use
our helper and you are good to go.

:::tip[Task]
Now try adding a constructor for a `Service` to `kubernetes.libsonnet`
and use both helpers to recreate the other objects as well.
:::
