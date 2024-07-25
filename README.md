# About

Use `secretGenerator` with `SealedSecrets` by teaching Kustomize about the relationship between SealedSecrets and Secrets names

I find myself repeating this in Kustomizations a lot, so here it is in a reusable form.

# Usage

Add the following to the Kustomization:

```diff
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

+ configurations:
+  - https://raw.githubusercontent.com/LCOGT/kustomize-sealedsecrets-config/main/config.yaml
```

This will let you use a `secretGenerator` with `SealedSecrets`.

For example, say you have `Pod` like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: something
      image: some-image
      envFrom:
        - secretRef:
            name: test
```

And you want that `secretRef` to point to a `Secret` generated and managed by a `SealedSecret`. That would look something like:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: test # <--- Same as secretRef.name
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true" # <--- Required because the name is generated and will change
spec:
  encryptedData:
    a: "1...."
    b: "2..."
    c: "3..."
  template:
    metadata:
      name: test
    type: Opaque
```

Then, you could use a `secretGenerator` like this in your `kustomization.yaml`:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configurations:
  - https://raw.githubusercontent.com/LCOGT/kustomize-sealedsecrets-config/main/config.yaml

resources:
  - ./pod.yaml
  - ./sealedsecret.yaml

secretGenerator:
  - name: test
    files:
      - ./sealedsecret.yaml # <--- The name will change if the content of the SealedSecret changes
    options:
      annotations:
        config.kubernetes.io/local-config: "true" # <--- This excludes this Secret from the output
```

To get a `kustomize build` output of:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  name: test-cm5hd7d6b9
spec:
  encryptedData:
    a: 1....
    b: 2...
    c: 3...
  template:
    metadata:
      name: test-cm5hd7d6b9
    type: Opaque
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - envFrom:
    - secretRef:
        name: test-cm5hd7d6b9
    image: some-image
    name: something
```
