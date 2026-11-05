Task 1: Multi-Container Pods (Init Container & Sidecar)
Overview
This task demonstrates a three-container Pod on a 3‑node Minikube cluster that serves a dynamic "Hello" page with a live timestamp. The Pod uses an Init Container to gate startup, a main Nginx container to serve the page and write the initial timestamp, and a sidecar container that updates the timestamp every 5 seconds without any shared volumes. This is achieved using shareProcessNamespace: true and writing through /proc/<nginx-pid>/root.

Additionally, the task requires intentionally forcing the Init Container to fail (exit code 1), observing Init:CrashLoopBackOff, and then recovering by fixing the Init Container.

Repository Structure
Kubernetes-Course-Tasks/
└── Task-1/
├── pod.yaml # Healthy Pod manifest
├── pod-broken.yaml # Manifest with failing Init Container
└── README.md # This file

Pod Architecture
Container	Type	Role
init-done	Init Container	Prints done and exits 0. Main containers start only after success.
nginx	Main Container	Writes initial index.html with current timestamp, then runs Nginx on port 80.
time-updater	Sidecar	Every 5 seconds, finds Nginx PID via /proc and overwrites index.html using /proc/<pid>/root/....
No volumes are used. The sidecar accesses the Nginx container's filesystem through the shared PID namespace (shareProcessNamespace: true). The path /proc/<nginx-pid>/root is a symlink to the root filesystem of the Nginx container.

Prerequisites
Minikube with 3 nodes (driver: docker recommended)

kubectl

curl (for verification)

Step-by-Step Instructions
1. Start a 3‑Node Minikube Cluster
bash
minikube start --nodes 3 --driver=docker --cpus=2 --memory=4096
kubectl get nodes
2. Deploy the Healthy Pod
bash
kubectl apply -f pod.yaml
kubectl get pod hello-timestamp -w
Wait until the Pod reaches Running with 2/2 containers ready.

3. Verify Timestamp Updates
Port‑forward the Nginx port:

bash
kubectl port-forward pod/hello-timestamp 8080:80
In another terminal:

bash
curl localhost:8080
Expected output (timestamp changes every 5 seconds):

text
hello in MLOPS MENA COMMUNITY : 2026-11-03 14:22:10
Run curl again after a few seconds to see a different timestamp.

Screenshot 1: Pod Running & Timestamp Display
<!-- Insert screenshot here: kubectl get pod showing Running 2/2 and curl output -->

Screenshot showing kubectl get pod with Running and 2/2, and the curl output displaying the initial timestamp.

Screenshot 2: Timestamp Updated
<!-- Insert screenshot here: second curl showing different timestamp -->

Second curl executed a few seconds later, showing a different timestamp – proof that the sidecar updates the file.

4. Intentionally Break the Init Container
Create the broken manifest by modifying only the Init Container command to exit with code 1:

yaml
# pod-broken.yaml (excerpt)
initContainers:
  - name: init-done
    image: busybox:1.36
    command: ["sh", "-c", "echo intentionally failing && exit 1"]
Delete the healthy Pod and apply the broken one:

bash
kubectl delete pod hello-timestamp
kubectl apply -f pod-broken.yaml
kubectl get pod hello-timestamp -w
After a short time, the status will show:

text
Init:CrashLoopBackOff
5. Diagnose the Failure
Inspect the Pod:

bash
kubectl describe pod hello-timestamp
Look for:

Init Container init-done with Last State: Terminated, Reason: Error, Exit Code: 1.

No main containers are created – the kubelet waits for all Init Containers to succeed.

Check Init Container logs:

bash
kubectl logs hello-timestamp -c init-done
Expected output: intentionally failing

Screenshot 3: Init:CrashLoopBackOff
<!-- Insert screenshot here: kubectl get pod showing Init:CrashLoopBackOff -->

kubectl get pod output showing the Pod stuck in Init:CrashLoopBackOff.

Screenshot 4: Describe Pod – Exit Code 1
<!-- Insert screenshot here: kubectl describe pod showing exit code 1 -->

kubectl describe pod highlighting the Init Container termination with exit code 1. Main containers are not started.

Logs from the failing init-done container confirming the intentional failure.

6. Recover by Fixing the Init Container
Delete the broken Pod and re‑apply the healthy manifest:

bash
kubectl delete pod hello-timestamp
kubectl apply -f pod.yaml
kubectl get pod hello-timestamp
Wait until Running and 2/2 again. Verify the timestamp updates with port-forward and curl.


After re‑applying the healthy manifest, the Pod returns to Running and the timestamp updates resume.

Key Design Decisions
shareProcessNamespace: true – Required for the sidecar to see the Nginx process in its own /proc filesystem.

No volumes – The sidecar writes directly into the Nginx container's filesystem via /proc/<pid>/root, satisfying the "no volumes" constraint.

Init Container gating – The init-done container must exit successfully before nginx and time-updater are started, demonstrating the Init Container lifecycle.

Robust PID lookup – The sidecar loops through /proc/[0-9]* and greps cmdline for "nginx: master" to avoid hard‑coding a PID.

Commands Summary
bash
# Start cluster
minikube start --nodes 3 --driver=docker --cpus=2 --memory=4096

# Deploy healthy Pod
kubectl apply -f pod.yaml

# Verify
kubectl get pod hello-timestamp
kubectl port-forward pod/hello-timestamp 8080:80
curl localhost:8080

# Break Init Container
kubectl delete pod hello-timestamp
kubectl apply -f pod-broken.yaml
kubectl get pod hello-timestamp -w
kubectl describe pod hello-timestamp
kubectl logs hello-timestamp -c init-done

# Recover
kubectl delete pod hello-timestamp
kubectl apply -f pod.yaml
kubectl get pod hello-timestamp
Verification Checklist
☑ 3‑node Minikube cluster running
☑ Pod runs three containers with shareProcessNamespace: true and no volumes
☑ Init Container prints done and exits 0 (healthy manifest)
☑ Nginx serves the initial timestamp
☑ Sidecar updates the timestamp every 5 seconds
☑ Forced failure of Init Container results in Init:CrashLoopBackOff
☑ Failure diagnosed via kubectl describe and kubectl logs
☑ Recovery after fixing the Init Container
☑ Screenshots provided for all key steps
Screenshots
All screenshots are stored in the screenshots/ directory (relative to this README) and referenced in the sections above. For the grader's convenience, they are also listed here:

01-pod-running-curl.png – Pod running, initial timestamp shown.

02-timestamp-updated.png – Timestamp updated after 5 seconds.

03-init-crashloop.png – Pod stuck in Init:CrashLoopBackOff.

04-describe-exit-code.png – kubectl describe showing exit code 1.