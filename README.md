# Guestbook on Kubernetes (IBM Cloud ICR)

A tiny Go web app (“Guestbook”) containerized, pushed to **IBM Cloud Container Registry (ICR)**, and deployed to Kubernetes with rolling updates, autoscaling, and rollback.

## What we built
- **Multi-stage Docker image** (Go builder → small Ubuntu runtime).
- Pushed to **us.icr.io/sn-labs-johndtwaldro/** as `guestbook:v1` and `guestbook:v2`.
- **Kubernetes Deployment** (container port 3000), port-forwarded for access.
- **HPA** (HorizontalPodAutoscaler) based on CPU (min=1, max=10, target=5%).
- **Rolling update** from v1 → v2 (UI text changed to “John’s Guestbook - v2”).
- **Rollback** back to v1 and verification via ReplicaSets & pod images.

## Key learnings
- **ICR basics**: `ibmcloud cr region-set us-south`, `ibmcloud cr login`, naming: `us.icr.io/<namespace>/<repo>:<tag>`.
- **Docker multi-stage** keeps runtime image small; copy the built binary with  
  `COPY --from=builder /app/main /app/guestbook`.
- **Deployment versioning**: change image tag and `kubectl set image` → watch with `kubectl rollout status` and audit with `kubectl rollout history`.
- **HPA can affect rollouts**: scale decisions may delay termination of old pods; pause/remove HPA during controlled rollouts if needed.
- **Port-forward gotcha**: “address already in use” → kill the old one  
  `pkill -f 'kubectl.*port-forward.*3000:3000'`.
- **Namespacing typos** matter (`johndtwaldro` vs `johndtwaldron`)—export once and reuse:  
  `export MY_NAMESPACE=sn-labs-johndtwaldro`.

## File structure
guestbook/
├─ main.go
├─ Dockerfile
├─ deployment.yml
├─ public/
│ ├─ index.html (v1 → v2 title/header change)
│ ├─ script.js
│ ├─ style.css
│ └─ jquery.min.js
└─ docs/screenshots/ (lab proofs: *.png)

bash
Copy code

## Build & Push
```bash
export MY_NAMESPACE=sn-labs-johndtwaldro
ibmcloud cr region-set us-south
ibmcloud cr login

# v1
docker build . -t us.icr.io/$MY_NAMESPACE/guestbook:v1
docker push us.icr.io/$MY_NAMESPACE/guestbook:v1

# v2 (after editing public/index.html)
docker build . -t us.icr.io/$MY_NAMESPACE/guestbook:v2
docker push us.icr.io/$MY_NAMESPACE/guestbook:v2
Deploy / Update / Access
bash
Copy code
# create/update deployment
kubectl apply -f deployment.yml
kubectl rollout status deploy/guestbook

# switch image to v2
kubectl set image deploy/guestbook \
  guestbook=us.icr.io/$MY_NAMESPACE/guestbook:v2
kubectl rollout status deploy/guestbook

# port-forward (local 3000 → pod 3000)
kubectl port-forward deployment.apps/guestbook 3000:3000
# if busy: pkill -f 'kubectl.*port-forward.*3000:3000'
Autoscaling
bash
Copy code
kubectl autoscale deployment guestbook \
  --cpu-percent=5 --min=1 --max=10
kubectl get hpa guestbook --watch
Tip: If a rollout seems stuck while HPA is scaling, temporarily disable it:

bash
Copy code
kubectl delete hpa guestbook --ignore-not-found
kubectl scale deploy/guestbook --replicas=1
kubectl rollout status deploy/guestbook
Rollback
bash
Copy code
# see revisions
kubectl rollout history deploy/guestbook
# inspect a specific revision
kubectl rollout history deploy/guestbook --revision=3
# rollback
kubectl rollout undo deploy/guestbook --to-revision=1
kubectl rollout status deploy/guestbook
Verification snippets
bash
Copy code
kubectl get rs
kubectl get pods -l app=guestbook \
  -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.containers[0].image}{'\n'}{end}"
kubectl describe deploy/guestbook | grep -i 'Image:'
Screenshots (for submission)
docs/screenshots/:

dockerfile.png

crimages.png

app.png

hpa.png / hpa2.png

upguestbook.png

deployment.png

rev.png

up-app.png
