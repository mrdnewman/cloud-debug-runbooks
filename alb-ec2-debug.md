# ==========================================
# ALB → EC2 DEBUG RUNBOOK
# ==========================================

FLOW
Client → ALB → Target Group → EC2 → Service → Endpoint


# -------------------------------------------------
# 1. Get ALB DNS
# -------------------------------------------------

aws elbv2 describe-load-balancers \
--query 'LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,ARN:LoadBalancerArn}' \
--output table

# copy ALB_DNS


# -------------------------------------------------
# 2. Test ALB
# -------------------------------------------------

curl -v http://ALB_DNS

# IF 200 → system OK
# IF 502 / 503 / 504 → backend problem


# -------------------------------------------------
# 3. Get Target Group ARN
# -------------------------------------------------

aws elbv2 describe-target-groups \
--query 'TargetGroups[*].{Name:TargetGroupName,ARN:TargetGroupArn}' \
--output table


# -------------------------------------------------
# 4. Check Target Health
# -------------------------------------------------

aws elbv2 describe-target-health \
--target-group-arn TARGET_GROUP_ARN \
--query 'TargetHealthDescriptions[*].{Instance:Target.Id,State:TargetHealth.State}' \
--output table

# copy INSTANCE_ID


# -------------------------------------------------
# 5. Get Instance Private IP
# -------------------------------------------------

aws ec2 describe-instances \
--instance-ids INSTANCE_ID \
--query 'Reservations[*].Instances[*].PrivateIpAddress' \
--output text


# -------------------------------------------------
# 6. SSH into instance
# -------------------------------------------------

ssh ubuntu@PRIVATE_IP
ssh ec2-user@PRIVATE_IP


# -------------------------------------------------
# 7. Check service
# -------------------------------------------------

systemctl status nginx
systemctl status httpd
systemctl status app

# IF stopped
sudo systemctl restart nginx
sudo systemctl restart httpd


# -------------------------------------------------
# 8. Check open ports
# -------------------------------------------------

ss -tulnp | grep LISTEN

# look for
# :80
# :8080


# -------------------------------------------------
# 9. CURL TESTS (endpoint checks)
# -------------------------------------------------

curl localhost
curl localhost:80
curl localhost:8080
curl localhost/
curl localhost/health
curl -I localhost

# RESULTS

# if NONE work
# → service down / wrong port / crash

# if port works but /health fails
# → health check mismatch


# -------------------------------------------------
# 10. Fix Health Check Path
# -------------------------------------------------

# check current path

aws elbv2 describe-target-groups \
--target-group-arns TARGET_GROUP_ARN \
--query 'TargetGroups[*].HealthCheckPath'


# modify health check path

aws elbv2 modify-target-group \
--target-group-arn TARGET_GROUP_ARN \
--health-check-path /


# or change to /health depending on app


# -------------------------------------------------
# 11. If LOCAL works but ALB fails
# -------------------------------------------------

# check target group port

aws elbv2 describe-target-groups \
--target-group-arns TARGET_GROUP_ARN \
--query 'TargetGroups[*].Port'


# if port mismatch
# update target group port


# -------------------------------------------------
# 12. Security Group Fix
# -------------------------------------------------

# get instance SG

aws ec2 describe-instances \
--instance-ids INSTANCE_ID \
--query 'Reservations[*].Instances[*].SecurityGroups[*].GroupId'


# allow ALB SG to reach instance port

aws ec2 authorize-security-group-ingress \
--group-id INSTANCE_SG \
--protocol tcp \
--port 80 \
--source-group ALB_SG


# -------------------------------------------------
# 13. Logs
# -------------------------------------------------

journalctl -u nginx -n 50
journalctl -xe

tail -f /var/log/nginx/error.log
