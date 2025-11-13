kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443

Login:

Username: admin
Password:

kubectl get secret argocd-initial-admin-secret  -n argocd  -o jsonpath="{.data.password}" | base64 -d   ====> IVyzmjIectQnGgUn


kubectl patch application laravel -n argocd --type='merge' -p='{"spec":{"source":{"directory":null,"kustomize":{"images":[]}}}}'





================ aws setup ======


1️⃣ Update kubeconfig to point to your EKS cluster
aws eks --region us-east-1  update-kubeconfig --name laravel-eks-cluster --profile personal-emam


<region> → your EKS cluster region (e.g., us-east-1)

<cluster-name> → the name of your EKS cluster

<aws-profile> → optional if you have multiple profiles (e.g., personal-emam)

This command adds the EKS cluster to your ~/.kube/config.

2️⃣ Check the current context
kubectl config get-contexts
kubectl config current-context


You should see eks-<cluster-name> as the current context.

If not, switch manually:

kubectl config use-context <eks-context-name>

3️⃣ Verify connection to EKS
kubectl get nodes

https://E854D8988C97DC656F9401AA80B765E8.gr7.us-east-1.eks.amazonaws.com

You should see your EKS worker nodes, not Minikube nodes.

4️⃣ Now create Argo CD namespace on EKS
kubectl create namespace argocd