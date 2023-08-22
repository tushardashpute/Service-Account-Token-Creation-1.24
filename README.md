Issue

Starting Kubernetes 1.24, Secrets are not automatically generated when Service Accounts are created.
Pods still have a token inside by default belonging to their ServiceAccount (application using SA will not face any issue)
For RBAC we are creating SA separately there we will face the issue, as Token is getting created auto only if it is used by Pod on the same cluster.




Current ServiceAccount creation
How Service account is getting created before 1.22




$ kubectl create sa testsa
serviceaccount/testsa created
$ kubectl get sa testsa -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-08-10T09:37:01Z"
  name: testsa
  namespace: utilities
  resourceVersion: "575187584"
  uid: 525907b3-1f30-45db-83ad-3ea0e02fa762
secrets:
- name: testsa-token-8hwqr




Here the secret is getting auto-created with Service Account.

$ kubectl get sa testsa        
NAME     SECRETS   AGE
testsa   1         3m10s
$ kubectl get secret testsa-token-8hwqr 
NAME                 TYPE                                  DATA   AGE
testsa-token-8hwqr   kubernetes.io/service-account-token   3      4m24s




Now this token will be permanent and we can use it for RBAC externally or the application will automount it as per requirement.




ServiceAccount creation in 1.24
What happens when you create a ServiceAccount in Kubernetes 1.24 ?




$ kubectl create sa testnew
serviceaccount/testnew created

$ kubectl get sa testnew
NAME      SECRETS   AGE
testnew   0         2m42s

$ kubectl get sa testnew -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-08-10T09:37:38Z"
  name: testnew
  namespace: utilities
  resourceVersion: "4002175"
  uid: 9fce82c0-cc83-417d-af3e-7e410b136524




As indicated in above, a secret is not automatically generated when a ServiceAccount is created.




apiVersion: apps/v1
kind: Deployment
metadata:
  name: sa-test
  labels:
    name: sa-test
  namespace: utilities
spec:
  replicas: 1
  selector:
    matchLabels:
      name: sa-test
  template:
    metadata:
      labels:
        name: sa-test
    spec:
      containers:
      - name: sa-test
        image: brentley/awscli
        command: ["sleep", "3600"]
      serviceAccountName: testnew




The above deployment is referencing the newly created testnew service account.

Lets inspect the pod and see whats mounted in it.




kubectl exec -it sa-test-86d46fd856-qm89l -- sh
/ # cat var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6ImMzYjM2YzU3ZTk5YjViNTkyMjUxNTEwYmJhYWZmYjE4ZGZlODlmOTkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIl0sImV4cCI6MTcyMzE5ODAwMywiaWF0IjoxNjkxNjYyMDAzLCJpc3MiOiJodHRwczovL29pZGMuZWtzLnVzLWVhc3QtMS5hbWF6b25hd3MuY29tL2lkLzY3MTQ3NzMzOEU4Q0M1RkM2NEU1Mjc2NTBERjlGOUExIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJ1dGlsaXRpZXMiLCJwb2QiOnsibmFtZSI6InNhLXRlc3QtODZkNDZmZDg1Ni1xbTg5bCIsInVpZCI6IjZiMjFiNzAzLTBiNjUtNDY0Yy05NzZlLTA4MmExYjExZGUzMyJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoidGVzdG5ldyIsInVpZCI6IjlmY2U4MmMwLWNjODMtNDE3ZC1hZjNlLTdlNDEwYjEzNjUyNCJ9LCJ3YXJuYWZ0ZXIiOjE2OTE2NjU2MTB9LCJuYmYiOjE2OTE2NjIwMDMsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp1dGlsaXRpZXM6dGVzdG5ldyJ9.IUYxKFUN0YJov1Z0SCmDh0h6gFk8v8cXycVGel0gOUn2CaJzYByu67k7DpG7rsl3G95IcPb2WcFZqhV214niG-u_ck9ZimffchAbtCuVV_6whH94tceAEqhCw14SXjHtAIDJzkz_qWDopfZSgPisxKweWBtN1Z0Wpd9KL50mEWD4Fgsgt46dE-IDAO79oUzQKzQqtV7vjYb3KI3PpJy2-gwKWMuhg0PhvF06ZCoatP3R9TM9Y2FZ3bR2r-2_12gcnLUsHaLPLnE-8JtGzZsNyFKMQzAPeGxKub0-sAWkMpNmRZiVQnIhsU58fKnw2b5BaxMS8c6H_v_t1cFHwVqzIw




Nothing changes when a Pod is created — a token is generated and automatically mounted into the Pod.

How to create ServiceAccount Secrets for External Applications to access the Kubernetes API?
Create a secret with the type service-account-token and pass the service-account name in the annotation section as shown below.




apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
 name: testnew-secret
 annotations:
   kubernetes.io/service-account.name: testnew




How to identify which token is associated to which ServiceAccount ?




Name:         testnew-secret
Namespace:    utilities
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: testnew
              kubernetes.io/service-account.uid: 9fce82c0-cc83-417d-af3e-7e410b136524

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  9 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IllKMWVQTW1Yd2VLUWVTekgxcVhONFRFZ2dUWkNsaHAtU09TMXJPUGJTS3cifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ1dGlsaXRpZXMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoidGVzdG5ldy1zZWNyZXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidGVzdG5ldyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjlmY2U4MmMwLWNjODMtNDE3ZC1hZjNlLTdlNDEwYjEzNjUyNCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp1dGlsaXRpZXM6dGVzdG5ldyJ9.IKp3IPBcKK0uQj_o68_pK9lvtATo9anQ_V9JbdOFIstDUbSwTXWNfqK5hPB5HBMxrOSeiXh-rEYVsUwHNvcJDPfr6nXyAA4HQZQhhNnZv2XlDqR1qqzzUd82JlsiIihgv11XPIsRtSDhdgq67duS1XRrhyV9Ysw0zeOjSdhwiYjmO8smT4mZW-dkUypn6ulMKkBwJ0m7zlQ5HGD1XytsAcYRs6wFOR-FhwAZRB3ERAw_fS7l2LXZ_IpPcvbSwuiNFv4CZdjCKfnApWFdhz-PYYStZTeKI9hpZcOA1AdJ4ggcEol-l5gaEpyp9frZ7quGzlQAWnMcS9DOzAP5SKusRg






RBAC fix in current helm chart

Earlier we were only creating Service Account with a helm chart but now added code to create a Secret as well and annotate it with the respective Service Account to have the same behavior as previously.
