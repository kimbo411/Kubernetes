# 1. Renovación De Certificados Kubernetes version 1.15.3
Estos pasos se realizan en todos los Master del cluster

* Generamos copias de seguridad de manifests
```bash
mkdir /etc/kubernetes/old2
cp /etc/kubernetes/manifests/*.* /etc/kubernetes/old2/
```
* Eliminamos los pods estaticos de kubernetes
```bash
rm -f /etc/kubernetes/manifests/*.*
```

* Generamos copias de seguridad de los certificados en la ruta /etc/kubernetes/k8s-old-certs2/pki

```bash
mkdir -p /etc/kubernetes/k8s-old-certs2/pki
/bin/cp -p /etc/kubernetes/pki/*.* /etc/kubernetes/k8s-old-certs2/pki
ls -l /etc/kubernetes/k8s-old-certs2/pki/
```

* Eliminamos los certificados
```bash
/bin/rm -f /etc/kubernetes/pki/apiserver.key
/bin/rm -f /etc/kubernetes/pki/apiserver.crt
/bin/rm -f /etc/kubernetes/pki/apiserver-kubelet-client.crt
/bin/rm -f /etc/kubernetes/pki/apiserver-kubelet-client.key
/bin/rm -f /etc/kubernetes/pki/front-proxy-client.crt
/bin/rm -f /etc/kubernetes/pki/front-proxy-client.key
```

* Regeneramos los certificados del controll-plane
```bash
kubeadm init phase certs all --config /etc/kubernetes/kubeadm-config.yaml
```

* Eliminamos archivos de configuración
```bash
/bin/rm -f /etc/kubernetes/admin.conf
/bin/rm -f /etc/kubernetes/kubelet.conf
/bin/rm -f /etc/kubernetes/controller-manager.conf
/bin/rm -f /etc/kubernetes/scheduler.conf 
```

* Regeneramos los archivos de configuración
```bash
kubeadm init phase kubeconfig all --config /etc/kubernetes/kubeadm-config.yaml
```


* Colocamos los archivos manifest (pods estaticos) en su lugar
```bash
cp /etc/kubernetes/old2/*.* /etc/kubernetes/manifests/
```

* Verificamos que los pods estáticos se inicien
```bash
docker ps
```

* Copiamos el archivo de configuración para acceder a la API por kubectl
```bash
cp -f /etc/kubernetes/admin.conf ~/.kube/config
```

* Reiniciamos el servicio de kubelet
```bash
systemctl daemon-reload&&systemctl restart kubelet
```

* Verificar vencimiento de certificado (apiserver)
```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep ' Not '
```

* Verificamos los nodos que estén en Ready
```bash
watch kubectl get nodes
```

* Verificamos los pods
```bash
watch kubectl get po -A
```

# Si los nodos no están en Ready, aplicar lo siguiente:
* Crear un token en el master y compiamos el resultado, para reemplazar en cada nodo:
```bash
kubeadm token create --ttl 24h0m0s
```

* Ingresamos en cada nodo (worker), reeplazar "< TOKEN >" por el copiado en el paso anterior:
```bash
new_token=<TOKEN>
sed -i "s/token: .*/token: $new_token/" /etc/kubernetes/bootstrap-kubelet.conf
systemctl daemon-reload
systemctl restart kubelet
```

