# Book Review App on Kubernetes

Author: Kingsley Erhatiemwonmon
Platform: Minikube (Calico CNI, containerd), namespace book-review

## 1. Architecture

Browser -> frontend Service (LoadBalancer :80) -> Nginx (:80) in the frontend Pod
- "/" goes to Next.js (:3000, same Pod)
- "/api/" goes to the backend Service (ClusterIP :3001) -> backend Pods -> mysql Service (:3306) -> MySQL StatefulSet -> PVC data-mysql-0

| Tier | Objects |
|------|---------|
| Presentation | Deployment frontend (2 replicas: nginx + nextjs containers), Service frontend (LoadBalancer, 80) |
| Application | Deployment backend (2 replicas, Express on 0.0.0.0:3001), Service backend (ClusterIP, 3001) |
| Data | StatefulSet mysql (1 replica, MySQL 8.0), headless Service mysql (3306), PVC data-mysql-0 (1Gi) |
| Config | ConfigMaps book-review-config and nginx-config, Secret book-review-secret |
| Security | NetworkPolicies enforced by Calico |

## 2. Platform and images

- Minikube started with: minikube start --cni=calico --driver=docker
- Context used for all commands: minikube (a separate kind cluster on the machine was not used)
- Images built locally and loaded with "minikube image load": book-review-backend:1.0 and book-review-frontend:1.0 (built with NEXT_PUBLIC_API_URL=/api)
- Public images: mysql:8.0, nginx:1.27-alpine

## 3. Access method

Local cluster, so no public endpoint. The frontend Service is type LoadBalancer, but Minikube shows EXTERNAL-IP as pending without "minikube tunnel". The app was accessed with:

    kubectl port-forward svc/frontend 8080:80
    open http://localhost:8080

Port-forwarding does not prove public LoadBalancer access. The backend and MySQL Services have no external IP.

## 4. Configuration

- ConfigMap book-review-config: DB_HOST, DB_NAME, DB_DIALECT, PORT, ALLOWED_ORIGINS
- ConfigMap nginx-config: default.conf, mounted into the Nginx container
- Secret book-review-secret: DB_USER, DB_PASS, JWT_SECRET, MYSQL_ROOT_PASSWORD
- The real Secret was generated with random values, kept in manifests/secret.yaml, and is git-ignored. Only secret.example.yaml (placeholders) is committed.
- The backend does not receive the MySQL root password.

## 5. Network restrictions

- default-deny-all: denies all ingress and egress in the namespace
- allow-dns: allows DNS egress
- frontend-policy: ingress on port 80, egress only to backend on 3001
- backend-policy: ingress only from tier=frontend on 3001, egress only to tier=database on 3306
- mysql-policy: ingress only from tier=backend on 3306

Tests: a Pod in the default namespace was denied (wget timed out). A Pod labeled tier=frontend received the books JSON. The browser flow kept working.

## 6. Issues and fixes

1. NEXT_PUBLIC_API_URL is baked in at build time, so a ConfigMap cannot change it. Fix: build with NEXT_PUBLIC_API_URL=/api.
2. Route mismatch: the backend serves /api/..., api.js calls /books, and page.js calls /api/books (which becomes /api/api/books). Fix: Nginx rewrite ^/api/api/(.*)$ /api/$1.
3. The backend defaults to port 5000. Fix: PORT=3001 in the ConfigMap (containerPort alone does not change it).
4. app.listen had no host. Fix: bind to 0.0.0.0 (see app-changes/backend/server-js-change.md).
5. The backend folder had a .env and no .dockerignore. Fix: added .dockerignore files to backend and frontend.
6. The frontend Dockerfile ran npm run dev. Fix: multi-stage production Dockerfile with an ARG for the API URL.
7. Image build failed with a buildkit 404 after "minikube docker-env" (containerd is not Docker). Fix: ran "minikube docker-env -u", built with normal Docker, then ran "minikube image load".
8. Seed-data race between backend replicas. Fix: deployed 1 replica first, then scaled to 2.
9. CORS rejects POSTs from unknown origins. Fix: ALLOWED_ORIGINS=http://localhost:8080.
10. The upstream repo tracks .env and node_modules. Fix: the submission repo holds only manifests and app changes.
11. Seeded users have fake passwords. Fix: registered a new user to test login.

## 7. Validation

- Homepage loads and lists the books at http://localhost:8080
- curl http://localhost:8080/api/books returns JSON through Nginx, the backend Service, and MySQL
- Registered a user, logged in, and submitted a review that appeared in the Reviews table
- Backend restart (kubectl rollout restart): the app recovered and the review remained
- MySQL Pod restart (kubectl delete pod mysql-0): the PVC stayed Bound and the review survived

## 8. Deploy steps

    kubectl create namespace book-review
    kubectl config set-context --current --namespace=book-review
    kubectl create secret generic book-review-secret -n book-review --from-literal=DB_USER=<user> --from-literal=DB_PASS=<password> --from-literal=JWT_SECRET=<secret> --from-literal=MYSQL_ROOT_PASSWORD=<root-password>
    kubectl apply -f manifests/configmap.yaml
    kubectl apply -f manifests/mysql.yaml
    kubectl apply -f manifests/backend.yaml
    kubectl apply -f manifests/nginx-configmap.yaml
    kubectl apply -f manifests/frontend.yaml
    kubectl apply -f manifests/networkpolicy.yaml
    kubectl port-forward svc/frontend 8080:80

Start the backend with 1 replica, then scale to 2 after the seed log lines appear.

## 9. Not implemented

Optional Task 9 (Helm, HPA, TLS) and the optional dashboard screenshot were skipped.
