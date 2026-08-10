# Tekton CI Setup — DC-DR GitOps Project

## 1. Install catalog tasks (one-time, on minikube-dc)
kubectl config use-context minikube-dc
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.6/kaniko.yaml

## 2. Apply custom tasks
kubectl apply -f tekton/tasks/generate-tag.yaml
kubectl apply -f tekton/tasks/update-gitops-values.yaml

## 3. Apply the three pipelines
kubectl apply -f tekton/pipelines/java-ci-pipeline.yaml
kubectl apply -f tekton/pipelines/python-ci-pipeline.yaml
kubectl apply -f tekton/pipelines/angular-ci-pipeline.yaml

## 4. Create required secrets

### a) Git credentials (for git-clone AND git push-back from update-gitops-values task)
kubectl create secret generic git-credentials \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=<your-github-username> \
  --from-literal=password=<your-github-personal-access-token>

kubectl annotate secret git-credentials tekton.dev/git-0=https://github.com

### b) Harbor DC docker credentials (for kaniko push)
kubectl create secret docker-registry harbor-docker-credentials \
  --docker-server=192.168.58.2:30002 \
  --docker-username=admin \
  --docker-password=Harbor12345

## 5. Trigger a pipeline run manually (test)
kubectl create -f tekton/pipelineruns/java-ci-pipelinerun.yaml
tkn pipelinerun logs -f -n default --last

## Notes
- Update repo-url in each pipelinerun file with your actual GitHub repo URL before running.
- Harbor is HTTP (no TLS) in this lab setup — kaniko EXTRA_ARGS already includes --insecure --skip-tls-verify.
- The update-gitops-values task commits the new image tag back to charts/<app>/values.yaml — this is what ArgoCD (DC & DR) will watch to trigger deployment.
