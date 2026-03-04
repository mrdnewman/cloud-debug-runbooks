# ALB → EC2 DEBUG RUNBOOK

Flow:
Client → ALB → Target Group → EC2 → Service → Endpoint


# 1. Is ALB responding?
curl -v http://ALB_DNS

# IF 200 OK
# → system working

# IF 502 / 503 / 504
# → backend issue → check target health


# 2. Check target health
aws elbv2 describe-target-health --target-group-arn TARGET_GROUP_ARN

# IF unhealthy
# → ALB cannot reach instance properly
# → check service / port / health endpoint

# IF healthy
# → ALB can reach instance
# → application behavior or logs issue


# 3. Get instance IP
aws ec2 describe-instances --instance-ids INSTANCE_ID \
--query 'Reservations[*].Instances[*].PrivateIpAddress' \
--output text


# 4. SSH into instance
ssh ubuntu@PRIVATE_IP
# or
ssh ec2-user@PRIVATE_IP


# 5. Is service running?
systemctl status nginx
systemctl status app

# IF inactive / failed
sudo systemctl restart nginx
sudo systemctl restart app


# 6. Is application listening on a port?
ss -tulnp

# LOOK FOR
# 0.0.0.0:80
# 0.0.0.0:8080

# Example expected output
# LISTEN 0 128 0.0.0.0:80 users:(("nginx",pid=1234))


# IF no listening port
# → application not running → check logs


# 7. Does endpoint respond locally?
curl localhost
curl localhost:80
curl localhost:8080
curl localhost/health

# IF none respond
# → app not serving traffic
# → check logs or restart service


# 8. Check logs
journalctl -u nginx -n 50
journalctl -u app -n 50
tail -f /var/log/nginx/error.log


# 9. Health check validation
curl localhost/health

# IF /health fails but / works
# → ALB health check path mismatch
# → update health check path


# 10. If local curl works but ALB fails
# → network / config issue

# Check security group
aws ec2 describe-security-groups --group-ids SG_ID

# Ensure ALB SG can access port 80 / 8080
