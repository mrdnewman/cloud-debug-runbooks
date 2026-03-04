# ==========================================
# KUBERNETES INCIDENT DEBUG RUNBOOK
# ==========================================

# Core flow
# Node → Pod → Container → App → Resources → Service


# -------------------------------------------------
# 1. Identify failing workload
# -------------------------------------------------

kubectl get pods -A -o wide

# locate failing pod
# copy POD_NAME and NAMESPACE



# =================================================
# STATUS = Running
# =================================================

# pod healthy
# if app still failing → check service / ingress

kubectl get svc -n NAMESPACE
kubectl get endpoints SERVICE_NAME -n NAMESPACE

# if endpoints empty → label mismatch

kubectl get pods --show-labels -n NAMESPACE
kubectl describe svc SERVICE_NAME -n NAMESPACE

# FIX
kubectl edit svc SERVICE_NAME -n NAMESPACE
# or
kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = CrashLoopBackOff
# =================================================

# container repeatedly crashing

kubectl describe pod POD_NAME -n NAMESPACE

kubectl logs POD_NAME -n NAMESPACE
kubectl logs POD_NAME -n NAMESPACE --previous


# ---- If logs show missing environment variable ----

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# add env variable

# env:
# - name: VARIABLE_NAME
#   value: VALUE

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# ---- If logs show missing configmap ----

kubectl get configmap -n NAMESPACE

kubectl describe configmap CONFIGMAP_NAME -n NAMESPACE

kubectl edit configmap CONFIGMAP_NAME -n NAMESPACE

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# ---- If logs show missing secret ----

kubectl get secrets -n NAMESPACE

kubectl describe secret SECRET_NAME -n NAMESPACE

kubectl create secret generic SECRET_NAME \
--from-literal=KEY=VALUE \
-n NAMESPACE

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# ---- If container command / startup wrong ----

kubectl get pod POD_NAME -n NAMESPACE -o yaml | grep command -A5

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = ImagePullBackOff / ErrImagePull
# =================================================

kubectl describe pod POD_NAME -n NAMESPACE

kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.containers[*].image}'; echo


# ---- Fix incorrect image tag ----

kubectl set image deployment/DEPLOYMENT_NAME \
CONTAINER_NAME=IMAGE:TAG \
-n NAMESPACE

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# ---- Fix private registry authentication ----

kubectl get secrets -n NAMESPACE

kubectl create secret docker-registry regcred \
--docker-server=REGISTRY_URL \
--docker-username=USERNAME \
--docker-password=PASSWORD \
--docker-email=EMAIL \
-n NAMESPACE

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# add under spec:
# imagePullSecrets:
# - name: regcred

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = OOMKilled
# =================================================

# container exceeded memory limit

kubectl describe pod POD_NAME -n NAMESPACE | grep -i oom

kubectl get pod POD_NAME -n NAMESPACE -o yaml | grep -A5 resources


# resources structure

# resources:
#   requests:
#     memory: 256Mi
#   limits:
#     memory: 512Mi


# FIX increase memory

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# change example

# resources:
#   requests:
#     memory: 512Mi
#   limits:
#     memory: 1Gi


kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE

kubectl rollout status deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = Pending
# =================================================

# pod cannot be scheduled

kubectl describe pod POD_NAME -n NAMESPACE

# look for
# Insufficient cpu
# Insufficient memory
# node selector mismatch


# inspect cluster capacity

kubectl get nodes
kubectl describe node NODE_NAME

kubectl top nodes
kubectl top pods -A


# ---- Fix resource requests ----

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# reduce requests example

# resources:
#   requests:
#     cpu: 200m
#     memory: 256Mi


kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# FINAL VERIFICATION
# =================================================

kubectl get pods -n NAMESPACE -o wide

kubectl rollout status deployment DEPLOYMENT_NAME -n NAMESPACE

kubectl get endpoints SERVICE_NAME -n NAMESPACE
