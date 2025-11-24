# Quick Reference Guide – AWS CLI, CDK, and kubectl

## AWS CLI – Essentials

### Profiles
```bash
aws configure list-profiles
aws configure --profile <name>
aws sso login --profile <name>
aws sts get-caller-identity --profile <name>
```

### S3
```bash
aws s3 ls
aws s3 ls s3://bucket
aws s3 cp file.txt s3://bucket/
aws s3 sync ./dist s3://bucket/ --delete
```

### EC2
```bash
aws ec2 describe-instances
aws ec2 describe-instances --filters Name=tag:Name,Values=MyServer
```

### IAM
```bash
aws iam list-users
aws iam list-roles
aws iam get-role --role-name <name>
```

### CloudFormation
```bash
aws cloudformation describe-stacks
aws cloudformation delete-stack --stack-name <name>
```

### ECR
```bash
aws ecr get-login-password --region <reg>   | docker login --username AWS --password-stdin <acct>.dkr.ecr.<reg>.amazonaws.com

aws ecr describe-repositories
aws ecr list-images --repository-name <repo>
```

## AWS CDK – Essentials

### Bootstrap
```bash
cdk bootstrap aws://<account>/<region>
```

### Synthesize (Dry Run)
```bash
cdk synth
```

### Diff (Show Changes)
```bash
cdk diff
cdk diff --profile <profile>
```

### Deploy
```bash
cdk deploy
cdk deploy "MyStack" --profile <profile>
```

### Destroy
```bash
cdk destroy
```

### List stacks
```bash
cdk ls
```

## kubectl – Essentials

### Contexts & Config
```bash
kubectl config get-contexts
kubectl config use-context <context>
kubectl config view
```

### General
```bash
kubectl get pods
kubectl get pods -A
kubectl get svc
kubectl get deploy
kubectl get ingress
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Pod Debugging
```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl exec -it <pod> -- /bin/bash
```

### Apply / Replace / Delete
```bash
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl replace -f file.yaml
```

### Port Forward
```bash
kubectl port-forward <pod> 8080:80
```

### Restart Deployment
```bash
kubectl rollout restart deployment <name>
kubectl rollout status deployment <name>
```

### Edit
```bash
kubectl edit deployment <name>
kubectl edit configmap <name>
```

## Kustomize
```bash
kubectl kustomize overlays/dev
kubectl apply -k overlays/dev
```

## ArgoCD CLI
```bash
argocd login <argocd-url>
argocd app list
argocd app get <app>
argocd app sync <app>
argocd app delete <app> --cascade
```