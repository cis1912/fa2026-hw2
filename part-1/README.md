# Part 1: Ollama + Open WebUI on a Local Kind Cluster

So far you've run containers directly (`docker run`) and wired a few of them together with Docker Compose. Both of those approaches describe a single machine's worth of containers. Kubernetes takes a different approach: instead of telling it _how_ to start your containers, you describe the _desired state_ of your application — "I want one replica of this image, listening on this port, with this much storage" — and a control loop continuously works to make the real cluster match that description. If a container crashes, Kubernetes restarts it. If a node dies, Kubernetes reschedules the pod elsewhere. You never run `docker run` again; you hand Kubernetes a description and it keeps the world honest to it.

In this part, you'll deploy [Ollama](https://ollama.com/) (an LLM server) and [Open WebUI](https://github.com/open-webui/open-webui) (a ChatGPT-style frontend for it) onto a local Kubernetes cluster using [`kind`](https://kind.sigs.k8s.io/) ("Kubernetes IN Docker"). You'll hand-write the raw Kubernetes objects for Ollama yourself — a `Deployment`, a `Service`, and a `PersistentVolumeClaim` — so you feel the raw mechanics at least once; the Open WebUI side is already fully written for you, so you can focus on Ollama. In Part 2, you'll see how Helm lets you package all of this up and replicate it across environments.

## Step 0: Deployments and Services

> 🤔 This is something we'd like you to think about...

In [RESPONSE.md](../RESPONSE.md), write your response to the following question. The [Kubernetes concepts docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [Services docs](https://kubernetes.io/docs/concepts/services-networking/service/) will be useful here.

> What does a Deployment give you that a bare Pod doesn't? Why do we need a separate Service object instead of talking to a Pod's IP directly?

## Step 1: Creating the Cluster

Create a new cluster for this homework first! This is to just make sure that your existing environment doesn't interfere with this homework.

```bash
kind create cluster --name cis1912-hw2
```

This spins up a (new) single-node Kubernetes cluster running as a Docker container on your machine. Set your kubectl to use the new cluster by running:

```bash
kubectl config use-context kind-cis1912-hw2
```

Verify that your kubectl is looking at the new cluster:

```bash
kubectl cluster-info
```

## Step 2: Deploying Ollama

We've provided starter manifests in [`manifests/`](./manifests) with the structure sketched out and the interesting parts left as `# TODO`s:

- [`ollama-pvc.yaml`](./manifests/ollama-pvc.yaml) — a `PersistentVolumeClaim` so Ollama's downloaded models survive pod restarts.
- [`ollama-deployment.yaml`](./manifests/ollama-deployment.yaml) — a `Deployment` running the `ollama/ollama` image, with the PVC mounted where Ollama stores its data.
- [`ollama-service.yaml`](./manifests/ollama-service.yaml) — a `Service` giving the Deployment's pod a stable address other pods (and you, via port-forward) can reach.

Fill in the TODOs, then apply all three:

```bash
kubectl apply -f manifests/ollama-pvc.yaml -f manifests/ollama-deployment.yaml -f manifests/ollama-service.yaml
```

Watch the pod come up:

```bash
kubectl get pods -w
```

Once it reports `1/1 Running`, move on. If it's stuck, see the [Tips](#tips) section below.

## Step 3: Pulling a Model

A fresh Ollama server doesn't have any models loaded yet — you have to pull one, the same way you'd `docker pull` an image. Exec into the running pod and pull a small model:

```bash
kubectl exec -it deploy/ollama -- ollama pull qwen2.5:0.5b
```

This downloads straight into the pod, onto the PVC you mounted — so it'll still be there the next time the pod restarts. Once it finishes, verify the server works by port-forwarding its Service and querying the API directly:

```bash
kubectl port-forward svc/ollama 11434:11434
```

In another terminal:

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "What is kubernetes? Is it a fish?"
}'
```

You should see a stream of JSON chunks containing the model's response.

## Step 4: Deploying Open WebUI

Now let's put a frontend in front of Ollama. To keep this part focused on Kubernetes mechanics rather than yet more YAML-filling, we've already written the full [`webui-deployment.yaml`](./manifests/webui-deployment.yaml) and [`webui-service.yaml`](./manifests/webui-service.yaml) for you — no TODOs here, just apply them:

```bash
kubectl apply -f manifests/webui-deployment.yaml -f manifests/webui-service.yaml
```

Before you do, take a minute to actually read through `webui-deployment.yaml`. A couple of things worth noticing, since they'll matter in Part 2:

- `OLLAMA_BASE_URL` points at `http://ollama:11434` — the **Service** name you wrote in Step 2, not a Pod IP. This is exactly the point from Step 0: Services give you a stable address to depend on.
- `WEBUI_AUTH` is set to `"false"` so you don't have to create an account on first visit.

Open WebUI takes a little longer to start than Ollama does — give it a minute, and keep an eye on `kubectl get pods -w`. Once it's `Running`, port-forward it:

```bash
kubectl port-forward svc/ollama-webui 8080:8080
```

Visit `http://localhost:8080` in your browser. Open WebUI should already be configured to use your Ollama backend — start a chat with `qwen2.5:0.5b` and confirm you get a response.

## Reflection

Take a moment to notice what you just did:

- You applied five separate YAML files, in an order you had to remember (the PVC and Service needed to exist in roughly the right order relative to the Deployment referencing them) — and that's with the webui pair already written for you.
- You pulled the model by hand, inside a running pod. If you tore this whole deployment down and brought it back up from scratch, you'd have to remember to redo that step — nothing here automates it.
- Every name in these files — `ollama`, `ollama-data`, `ollama-webui` — is hardcoded. If you wanted a second, independent copy of this entire setup (say, one for "staging" and one for "production"), your only option right now is to copy every file and manually rename everything inside, twice.

In Part 2, you'll package this exact setup into a Helm chart, which fixes all three problems: one command installs everything in the right order, a hook automates the model pull, and the same chart can be installed multiple times under different names with different settings.

## Tips

- `kubectl describe pod <pod-name>` is your best first stop when a pod isn't reaching `Running` — it shows recent events (image pull failures, failed probes, scheduling issues) at the bottom of the output.
- `kubectl logs <pod-name>` shows what's happening inside the container itself; add `-f` to follow it live.
- `kubectl get events --sort-by=.lastTimestamp` across the whole cluster can surface problems that aren't obviously tied to one pod.
- `kubectl delete -f manifests/` tears everything in this part back down if you want to start over.
