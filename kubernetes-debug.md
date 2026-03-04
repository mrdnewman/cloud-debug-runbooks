# ==========================================
# KUBERNETES INCIDENT DEBUG RUNBOOK
# ==========================================

# -------------------------------------------------
# 1. See what is failing
# -------------------------------------------------

kubectl get pods -A


# =================================================
# STATUS = Running
# =================================================

# cluster / pod healthy
# nothing to fix



# =================================================
# STATUS = CrashLoopBackOff
# =================================================

# container repeatedly crashing

# inspect pod
kubectl describe pod POD_NAME -n NAMESPACE

# check logs
kubectl logs POD_NAME -n NAMESPACE

# if container restarted
kubectl logs POD_NAME -n NAMESPACE --previous

# fix usually requires:
# bad config
# missing env variable
# application error

# restart deployment after fix
kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = ImagePullBackOff / ErrImagePull
# =================================================

# image cannot be pulled

# inspect error
kubectl describe pod POD_NAME -n NAMESPACE

# check image being used
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='{.spec.containers[*].image}'; echo

# FIX OPTION 1: wrong image tag
kubectl set image deployment/DEPLOYMENT_NAME \
CONTAINER_NAME=IMAGE:TAG \
-n NAMESPACE

# restart rollout
kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE


# FIX OPTION 2: private registry authentication issue

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

# confirm OOM kill
kubectl describe pod POD_NAME -n NAMESPACE | grep -i oom

# check resource limits
kubectl get pod POD_NAME -n NAMESPACE -o yaml | grep -A5 resources

# FIX: increase memory limit

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# change section to something larger

# resources:
#   requests:
#     memory: 512Mi
#   limits:
#     memory: 1Gi

# restart deployment

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE

kubectl rollout status deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = Pending
# =================================================

# pod cannot be scheduled

kubectl describe pod POD_NAME -n NAMESPACE

# look for:
# Insufficient cpu
# Insufficient memory
# node selector issues

# check node resources
kubectl get nodes
kubectl describe node NODE_NAME

kubectl top nodes
kubectl top pods -A


# FIX OPTION 1: reduce resource requests

kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE

# lower requests:

# resources:
#   requests:
#     cpu: 200m
#     memory: 256Mi


# FIX OPTION 2: add cluster capacity
# (environment dependent)

# restart deployment after change

kubectl rollout restart deployment DEPLOYMENT_NAME -n NAMESPACE



# =================================================
# STATUS = Running but service unreachable
# =================================================

kubectl get svc -n NAMESPACE
kubectl get endpoints SERVICE_NAME -n NAMESPACE

# if endpoints empty → selector mismatch

kubectl get pods --show-labels -n NAMESPACE
kubectl describe svc SERVICE_NAME -n NAMESPACE

# fix selector or labels

kubectl edit svc SERVICE_NAME -n NAMESPACE
kubectl edit deployment DEPLOYMENT_NAME -n NAMESPACE
