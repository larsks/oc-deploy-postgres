This branch of the repository uses [ksops][] to manage secrets in the
repository.

## Requirements

Using ksops introduces three dependencies:

- You need to use [kustomize][] instead of `oc apply -k`.
- You need to install [sops][].
- You need to install the [ksops][] plugin.

[sops]: https://github.com/mozilla/sops
[ksops]: https://github.com/viaduct-ai/kustomize-sops
[kustomize]: https://kustomize.io/

## Deploying

Once you have the dependencies in place, deployment looks like this:

```
kustomize build --enable_alpha_plugins | oc apply -f-
```

And cleaning up looks like:


```
kustomize build --enable_alpha_plugins | oc delet -f-
```

## How does it work?

The `sops` tool uses GPG (or other key management systems) to encrypt
and decrypt files, with particular support for YAML documents. By
adding a `generator` to the `kustomization.yaml` file...


```
generators:
- secret-generator.yaml
```

...we instruct `kustomize` to use that file to generate additional
resources. The `secret-generator.yaml` file looks like this:


```
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
files:
  - postgres-secret.yaml
```

That instructs `kustomize` to use the `viaduct.ai/v1/ksops` plugin to
generate resources from the input file `postgres-ecret.yaml`. 
