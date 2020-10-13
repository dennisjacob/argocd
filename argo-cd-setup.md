### Argo CD Set up in On-Premise Kubernetes environment


Step 1:  Create a CA cert (Optional) and  TLS certtiifcate for configuring in ArgoCD 

```bash
root@kube-ub-master:~/argocd-certs# openssl genrsa -out argocd-ca.key
Generating RSA private key, 2048 bit long modulus (2 primes)
................+++++
...................................................................................................................+++++
e is 65537 (0x010001)

root@kube-ub-master:~/argocd-certs# openssl req -new -key argocd-ca.key -out argocd-ca.csr -subj "/CN=argocd-ca"

root@kube-ub-master:~/argocd-certs# openssl x509 -req -in argocd-ca.csr -signkey argocd-ca.key -out argocd-ca.crt
Signature ok
subject=CN = argocd-ca
Getting Private key

root@kube-ub-master:~/argocd-certs# openssl x509 -in argocd-ca.crt -subject -issuer -dates -noout
subject=CN = argocd-ca
issuer=CN = argocd-ca
notBefore=Oct  2 09:52:03 2020 GMT
notAfter=Nov  1 09:52:03 2020 GMT



root@kube-ub-master:~/argocd-certs# openssl genrsa -out argocd-server.k8s.local.key
Generating RSA private key, 2048 bit long modulus (2 primes)
.................................................+++++
....................................................+++++
e is 65537 (0x010001)

root@kube-ub-master:~/argocd-certs# openssl req -new -key argocd-server.k8s.local.key -out argocd-server.k8s.local.csr -subj "/CN=argocd-server.k8s.local"

root@kube-ub-master:~/argocd-certs# openssl x509 -req -in argocd-server.k8s.local.csr -CA argocd-ca.crt -CAkey argocd-ca.key -out argocd-server.k8s.local.crt  -CAcreateserial -CAserial serial
Signature ok
subject=CN = argocd-server.k8s.local
Getting CA Private Key

root@kube-ub-master:~/argocd-certs# openssl x509 -in argocd-server.k8s.local.crt -noout -subject -issuer -dates
subject=CN = argocd-server.k8s.local
issuer=CN = argocd-ca
notBefore=Oct  2 09:56:21 2020 GMT
notAfter=Nov  1 09:56:21 2020 GMT
root@kube-ub-master:~/argocd-certs#

```

Step 2: Installation of Argo CD

```bash
root@kube-ub-master:~/argocd-certs# kubectl create namespace argocd
namespace/argocd created

root@kube-ub-master:~/argocd-certs# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-secret created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server-metrics created
service/argocd-server created
deployment.apps/argocd-application-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
root@kube-ub-master:~/argocd-certs#

root@kube-ub-master:~/argocd-certs# kubectl get po -n argocd
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-7bb455497b-7jlkb   1/1     Running   0          49m
argocd-dex-server-567cff8446-bljnn               1/1     Running   2          49m
argocd-redis-99fb49846-pkjbx                     1/1     Running   0          49m
argocd-repo-server-64fd959984-z9zll              1/1     Running   0          49m
argocd-server-659c89647c-sjbsr                   1/1     Running   0          49m
root@kube-ub-master:~/argocd-certs#

```

Step3 :  Set up of Ingress

Set up the ingress in argocd namespace, with a TLS secret.
Application can be accessed using the https://argocd-server.k8s.local/ 


```bash

root@kube-ub-master:~/argocd-certs# kubectl create secret tls argo-tls-secret --cert=./argocd-server.k8s.local.crt --key=./argocd-server.k8s.local.key -n argocd
secret/argo-tls-secret created
root@kube-ub-master:~/argocd-certs# kubectl get secret -n argocd
NAME                                        TYPE                                  DATA   AGE
argo-tls-secret                             kubernetes.io/tls                     2      9s

root@kube-ub-master:~/argocd-certs# cat ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-argo-ingress
  namespace: argocd
spec:
  tls:
    - hosts:
        - argocd-server.k8s.local
      secretName: argo-tls-secret
  rules:
  - host: argocd-server.k8s.local
    http:
      paths:
      - path: /
        backend:
          serviceName: argocd-server
          servicePort: 443
root@kube-ub-master:~/argocd-certs# kubectl apply -f ing.yaml
ingress.extensions/my-argo-ingress created

root@kube-ub-master:~/argocd-certs# kubectl get ing -n argocd
NAME              CLASS    HOSTS                     ADDRESS        PORTS     AGE
my-argo-ingress   <none>   argocd-server.k8s.local   10.99.230.52   80, 443   32s
root@kube-ub-master:~/argocd-certs#

```

Step 4: An alternate option for Step3 is to use the NodePort, if you are using a lab on-premise set up.

```bash

#  Add a nodePort range to the kube-api server, that allows you to access non-ephimeral ports. Change in the static manifest will auto restart the kube-api server to pick up the new value

root@kube-ub-master:/etc/kubernetes/manifests# grep  service-node-port-range  /etc/kubernetes/manifests/kube-apiserver.yaml
    - --service-node-port-range=80-32767


# Modify the svc/argocd-server service to use node port. Using 8443 in this example

spec:
  clusterIP: 10.98.38.144
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 22420
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 8443
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: NodePort


# Check the argocd  service
root@kube-ub-master:~/argocd-certs# kubectl get svc/argocd-server -n argocd
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                     AGE
argocd-server   NodePort   10.98.38.144   <none>        80:22420/TCP,443:8443/TCP   173m



# Now, modify the argocd tls secret to modify the already prepared certificate in Step#1

Commands used to convert the certiricate and key to base64 encoded string:
cat argocd-server.k8s.local.crt | base64  | tr -d '\n'
cat argocd-server.k8s.local.key | base64  | tr -d '\n'


Update the secret/argo-tls-secret with the new key and cert and restart all deployments part of argocd

root@kube-ub-master:~/argocd-certs# for each in argocd-application-controller argocd-dex-server argocd-redis argocd-repo-server argocd-server; do kubectl rollout restart deploy/${each} -n argocd; done
deployment.apps/argocd-application-controller restarted
deployment.apps/argocd-dex-server restarted
deployment.apps/argocd-redis restarted
deployment.apps/argocd-repo-server restarted
deployment.apps/argocd-server restarted
root@kube-ub-master:~/argocd-certs#

#Verifying the new cert that is picked up by ArgoCD

root@kube-ub-master:~/argocd-certs# openssl s_client -connect kube-ub-master:8443 -showcerts | openssl x509 -noout -subject -issuer -dates
Can't use SSL_get_servername
depth=0 CN = argocd-server.k8s.local
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 CN = argocd-server.k8s.local
verify error:num=21:unable to verify the first certificate
verify return:1
subject=CN = argocd-server.k8s.local
issuer=CN = argocd-ca
notBefore=Oct  2 09:56:21 2020 GMT
notAfter=Nov  1 09:56:21 2020 GMT


```

Step5: Access the ArgoCD

Access the  ArgoCD console using
	- ArgoCD Server URL
	- ArgoCD APIs
	- ArgoCD CLI



	

