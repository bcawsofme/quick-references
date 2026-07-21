# Cloud Networking Quick Reference Guide

## DNS
```bash
dig example.com
dig A example.com
dig CNAME example.com
dig NS example.com
dig TXT example.com
nslookup example.com
host example.com
```

## TLS
```bash
openssl s_client -connect example.com:443 -servername example.com
openssl x509 -in cert.pem -text -noout
curl -v https://example.com
curl --resolve example.com:443:<ip> https://example.com
```

## HTTP Checks
```bash
curl -I https://example.com
curl -v https://example.com
curl -sS https://example.com/health
curl -w "%{http_code} %{time_total}\n" -o /dev/null -s https://example.com
```

## Linux Network Debugging
```bash
ip addr
ip route
ss -tulpn
ss -tan
nc -vz <host> <port>
traceroute <host>
mtr <host>
tcpdump -i any port 443
```

## AWS VPC
```bash
aws ec2 describe-vpcs
aws ec2 describe-subnets
aws ec2 describe-route-tables
aws ec2 describe-internet-gateways
aws ec2 describe-nat-gateways
aws ec2 describe-security-groups
aws ec2 describe-network-acls
```

## AWS Load Balancing
```bash
aws elbv2 describe-load-balancers
aws elbv2 describe-listeners --load-balancer-arn <arn>
aws elbv2 describe-target-groups
aws elbv2 describe-target-health --target-group-arn <arn>
aws route53 list-hosted-zones
aws route53 list-resource-record-sets --hosted-zone-id <zone-id>
```

## Kubernetes Networking
```bash
kubectl get svc -A
kubectl get ingress -A
kubectl get endpoints -A
kubectl get networkpolicy -A
kubectl describe svc <service>
kubectl describe ingress <ingress>
kubectl port-forward svc/<service> 8080:80
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
```

## Common Failure Checks
```bash
dig <hostname>
curl -vk https://<hostname>
nc -vz <hostname> 443
kubectl get endpoints <service>
kubectl describe ingress <ingress>
aws elbv2 describe-target-health --target-group-arn <arn>
```
