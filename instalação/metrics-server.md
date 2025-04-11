
### ✅ Como instalar o metrics-server no Kind

Você pode instalar usando o manifesto oficial **com um pequeno ajuste** (Kind usa certificados self-signed, então é preciso permitir isso).

---

### 📦 Passo a passo

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Depois disso, **edite o deployment** para permitir certificados inseguros:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Procure por `containers:` e adicione esse argumento dentro de `args:`:

```yaml
        - --kubelet-insecure-tls
```

O resultado vai ficar parecido com:

```yaml
containers:
- args:
  - --cert-dir=/tmp
  - --secure-port=4443
  - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --kubelet-insecure-tls   # <--- adicione isso
```

Salve e feche.

---

### ✅ Testando

Aguarde o pod subir:

```bash
kubectl get pods -n kube-system -l k8s-app=metrics-server -w
```

E então rode:

```bash
kubectl top nodes
kubectl top pods --all-namespaces
```
