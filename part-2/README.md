# Part 2: Packaging It With Helm

In Part 1, you hand-wrote three Kubernetes manifests for Ollama (plus the webui pair we handed you already finished), applied everything in the right order, pulled the model by hand inside the pod, and noticed that every name in those files was hardcoded — standing up a second, independent copy of this setup would mean copy-pasting and renaming everything.

[Helm](https://helm.sh/) is a package manager for Kubernetes that fixes exactly this. A Helm **chart** is a directory of templated manifests plus a `values.yaml` of defaults. Installing a chart creates a **release** — a named, tracked instance of that chart, with its own values. Install the same chart twice, under two different release names, with two different values files, and you get two independent deployments without maintaining two sets of YAML by hand. That's what you'll do here: turn your Part 1 manifests into a chart, then install it once for `staging` and once for `production`.

## Step 0: What Helm Buys You

> 🤔 This is something we'd like you to think about...(again)

In [RESPONSE.md](../RESPONSE.md), write your response to the following question. The [Helm charts docs](https://helm.sh/docs/topics/charts/) and [values docs](https://helm.sh/docs/chart_template_guide/values_files/) will be useful here.

> What is a Helm chart, and how do `values.yaml` plus `-f`/`--set` let the same chart produce different Kubernetes resources for different environments? Why is this better than keeping separate copies of your Part 1 manifests for staging and production?

## Step 1: Templatizing Part 1

We've scaffolded a chart at [`ollama/`](./ollama) with `Chart.yaml` and `values.yaml` already filled in. The `templates/` directory contains the same five objects from Part 1 as Helm templates, each left with `# TODO`s for you to templatize, the same way you hand-wrote them in Part 1:

- [`templates/pvc.yaml`](./ollama/templates/pvc.yaml)
- [`templates/deployment.yaml`](./ollama/templates/deployment.yaml)
- [`templates/service.yaml`](./ollama/templates/service.yaml)
- [`templates/webui-deployment.yaml`](./ollama/templates/webui-deployment.yaml)
- [`templates/webui-service.yaml`](./ollama/templates/webui-service.yaml)

Unlike Part 1, where we handed you a finished webui Deployment/Service, here you're templatizing *both* halves yourself — you already know exactly what they need to contain from reading through Part 1's version, so this is just applying the same Helm patterns a second time.

Two placeholders you'll use everywhere:

- `{{ .Release.Name }}` — the name you give this install (e.g. `ollama-staging`). Use it to prefix every object name, so two releases of this chart never collide.
- `{{ .Values.* }}` — pulls a field out of `values.yaml` (or whatever override file was passed with `-f`). This is what lets `-f values-staging.yaml` and `-f values-production.yaml` produce different resources from the same templates.

These TODOs are more open-ended than Part 1's `# TODO`s in places — rather than a blank sitting next to a given key, you'll sometimes need to add the key _and_ its nested structure yourself (a whole `containers:` block, a `ports:` list, and so on), the same way you did from scratch in Part 1. Each TODO still tells you which field in [`values.yaml`](./ollama/values.yaml) to pull from. The parts of each file that *are* already filled in — the `metadata` blocks, and `type`/`selector` in both Service files — are themselves a worked example of exactly how `{{ .Release.Name }}` and `{{ .Values.* }}` get used; lean on them.

## Step 2: Automating the Model Pull

In Part 1, you ran `kubectl exec ... -- ollama pull qwen2.5:0.5b` by hand after the pod came up — and you'd have to redo it on every fresh install. Helm supports [**hooks**](https://helm.sh/docs/topics/charts_hooks/): ordinary Kubernetes objects annotated to run at specific points in a release's lifecycle, instead of you remembering to run something by hand.

Fill in the TODOs in [`templates/job-pull-model.yaml`](./ollama/templates/job-pull-model.yaml) — a `Job` that waits for the Ollama server to come up and then pulls `.Values.model.name` into it, annotated to run automatically after install _and_ after upgrade.

## Step 3: Linting and Previewing

Before installing anything, check that the chart is well-formed and see what it would actually produce:

```bash
helm lint ollama
helm template ollama
```

`helm template` renders every manifest locally without touching your cluster — read through the output and make sure the names, images, and values look like what you'd expect. This is the fastest way to catch a templating mistake before it becomes a confusing `kubectl` error.

## Step 4: Installing to Staging

Install the chart under the release name `ollama-staging`, in its own namespace, using the staging overrides:

```bash
helm install ollama-staging ./ollama -n staging --create-namespace -f values-staging.yaml
```

Check that the model-pull hook ran:

```bash
kubectl get jobs -n staging
kubectl logs job/ollama-staging-pull-model -n staging
```

Then reach the WebUI the same way as Part 1:

```bash
kubectl port-forward svc/ollama-staging-ollama-webui 8080:8080 -n staging
```

Visit `http://localhost:8080` and confirm you can chat with the model — no manual pull step this time.

## Step 5: Installing to Production

Now install the _same chart_, unmodified, as a second release — a different name, a different namespace, and `values-production.yaml` instead:

```bash
helm install ollama-production ./ollama -n production --create-namespace -f values-production.yaml
```

Confirm both releases are running, completely independently, from the one chart:

```bash
helm list -A
kubectl get pods -n staging
kubectl get pods -n production
```

Port-forward production's WebUI on a different local port to compare side by side:

```bash
kubectl port-forward svc/ollama-production-ollama-webui 8081:8080 -n production
```

## Step 6: Upgrading One Without Touching the Other

Bump something in `values-staging.yaml` — for example, `resources.limits.memory` — and upgrade just that release:

```bash
helm upgrade ollama-staging ./ollama -n staging -f values-staging.yaml
```

Check `kubectl get pods -n production` again. Production should be completely unaffected — it's a separate release, tracking its own values independently, even though both came from the exact same chart on disk. This is the payoff: one chart, two (or more) environments, each independently deployable and upgradable.

## Cleanup

```bash
helm uninstall ollama-staging -n staging
helm uninstall ollama-production -n production
kubectl delete namespace staging production
kind delete cluster --name cis1912-hw2
```

## Tips

- `helm get values ollama-staging -n staging` shows exactly what values a running release was installed with — useful when you're not sure an override actually took effect.
- If a release gets stuck mid-install, `helm status <release> -n <namespace>` and `kubectl get jobs,pods -n <namespace>` will usually tell you whether it's the main Deployments or the pull-model hook Job that's stuck.
- `helm uninstall` removes a release's objects but not its namespace — you still need `kubectl delete namespace` if you want that gone too.
