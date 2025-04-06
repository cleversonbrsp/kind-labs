Cluster Kind com **1 control-plane e 1 worker node**, compatível com MetalLB para que seu `Service` com tipo `LoadBalancer` funcione corretamente.

---

## 🧱 Arquivo de configuração `kind-config.yaml` completo:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
      - containerPort: 443
        hostPort: 8443
  - role: worker
```

Esse config cria:
- 1 **control-plane** com mapeamento das portas 80 e 443 para sua máquina local.
- 1 **worker node** para simular melhor um ambiente real.

---

## 🛠️ Criar o cluster

```bash
kind create cluster --config kind-config.yaml --name kind-metallb
```

---

## 🌐 Verifique a rede do Kind (importante para o MetalLB)

```bash
docker network inspect kind | grep Subnet
```

Você verá algo como:

```json
"Subnet": "172.18.0.0/16",
```

A gente vai usar um range desse espaço, como: `172.18.255.200-172.18.255.250`

---

## 📦 Instalar o MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

---

## ⚙️ Configurar IP pool do MetalLB

Crie um arquivo chamado `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250  # <-- ajuste para combinar com a subnet
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advert
  namespace: metallb-system
```

Aplique com:

```bash
kubectl apply -f metallb-config.yaml
```

---

✅ **Agora seus Services do tipo LoadBalancer vão receber um IP!**

## Dica: evitar esses delays com kind load docker-image

**Se a imagem fake-shop for local (você buildou na sua máquina), o Kind não vê ela automaticamente, porque os nodes estão em containers separados.**

Use:
```bash
kind load docker-image thefly72003/fake-shop:v1 --name kind-metallb
```
Isso faz com que a imagem local seja "injetada" no Kind e ele não precise puxar da internet.
```bash
➜  fake-shop git:(main) ✗ kind get clusters
kind-metallb
➜  fake-shop git:(main) ✗ kind load docker-image thefly72003/fake-shop:v1 --name kind-metallb

Image: "thefly72003/fake-shop:v1" with ID "sha256:348874edcd4f7efe1295f67d5960fda189e5ccd26ac7d73ffd6118b22ee96a8f" not yet present on node "kind-metallb-control-plane", loading...
➜  fake-shop git:(main) ✗
```
