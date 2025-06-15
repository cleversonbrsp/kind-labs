### Implicações técnicas para rodar Envoy Gateway no kind + MetalLB

Vou explicar primeiro de forma simples com uma analogia, e depois ir subindo a complexidade com os detalhes técnicos e os ajustes práticos para rodar o **Envoy Gateway** num ambiente **kind + MetalLB**.

---

## 1. Explicação simples (analogia)

Imagine que você tem um prédio (seu cluster Kubernetes criado com kind), que não tem porta principal (não tem LoadBalancer “de fábrica”). MetalLB é como o porteiro que vai gerenciar as chaves (endereços IP externos) para que visitantes (tráfego externo) possam entrar no prédio.

Agora, o Envoy Gateway é uma portaria sofisticada dentro do prédio que recebe o visitante, decide para qual andar o visitante deve ir (roteia o tráfego HTTP/HTTPS para os serviços corretos) e aplica regras de segurança.

Para tudo funcionar:

* O porteiro (MetalLB) precisa estar lá para entregar a chave certa (IP externo) para a portaria (Envoy Gateway).
* A portaria (Envoy Gateway) precisa ser configurada para escutar no IP correto, que é fornecido pelo porteiro (MetalLB).
* A rede do prédio (kind) precisa permitir essa entrada e saída de tráfego.

---

## 2. Implicações técnicas para rodar Envoy Gateway no kind + MetalLB

### a) kind não oferece LoadBalancer nativo

* **kind** é um cluster Kubernetes local que roda dentro de containers Docker.
* Ele não tem suporte a LoadBalancer de forma nativa, porque não está em cloud pública.
* Por isso, usamos **MetalLB** para simular esse LoadBalancer, oferecendo IPs externos dentro de uma faixa configurada.

### b) MetalLB precisa ser configurado corretamente

* MetalLB deve ser instalado no cluster.
* Deve-se configurar o pool de IPs (range IPs) que MetalLB pode alocar.
* Esses IPs precisam ser acessíveis a partir do host (máquina local) — normalmente um range dentro da rede local (exemplo: 192.168.0.240-192.168.0.250).

### c) Envoy Gateway precisa ser exposto via Service LoadBalancer

* O Envoy Gateway roda como Deployment/Pod dentro do cluster.
* Para receber tráfego externo, seu Service deve ser do tipo `LoadBalancer`.
* Esse Service será “atrelado” a um IP externo fornecido pelo MetalLB.
* O Envoy escuta nesse IP para receber as requisições.

### d) Rede e roteamento no host local

* A máquina onde está o kind deve rotear o tráfego para o IP externo do MetalLB.
* Por isso, o range IP MetalLB deve estar na mesma rede do host, ou precisa de regras de roteamento.
* Caso contrário, seu browser ou curl não conseguem alcançar o IP.

### e) Configurações específicas do Envoy Gateway

* Configurar listeners, clusters e rotas do Envoy para expor corretamente os serviços.
* Configurar certificados TLS (se usar HTTPS).
* Ajustar timeout, retries, etc, conforme sua necessidade.

---

## 3. Ajustes práticos para rodar Envoy Gateway no kind + MetalLB

### Passo 1: Instalar e configurar MetalLB no kind

Exemplo de configuração básica para MetalLB (na faixa 192.168.0.240-192.168.0.250):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.240-192.168.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - my-ip-pool
```

Instale o MetalLB (se não fez ainda):

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.11/config/manifests/metallb-native.yaml
```

Depois aplique o IP pool e a L2Advertisement acima.

---

### Passo 2: Configurar o Service do Envoy Gateway

Exemplo mínimo do Service para expor o Envoy Gateway:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: envoy-gateway
  namespace: envoy-system
spec:
  selector:
    app: envoy-gateway
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  - protocol: TCP
    port: 443
    targetPort: 8443
```

Ao criar esse Service, MetalLB vai alocar um IP do pool configurado.

---

### Passo 3: Ajustar hosts e roteamento

* Certifique-se que o IP que MetalLB alocou está no mesmo segmento da sua rede local.
* Se estiver usando `kidimaix.local`, adicione no `/etc/hosts` apontando para esse IP.
* Se não conseguir pingar esse IP, o problema é de roteamento.

---

### Passo 4: Deploy do Envoy Gateway

* Configure o deployment do Envoy Gateway.
* Configure o Gateway API no Kubernetes para que o Envoy saiba quais hosts/rotas expor.

---

### Passo 5: Testes

* Acesse o IP externo do Service via navegador ou curl.
* Verifique logs e status do pod do Envoy Gateway.
* Ajuste firewall local, se houver bloqueios.

---

## 4. Resumo dos pontos críticos para funcionar

| Item                      | O que fazer                  | Por que                     |
| ------------------------- | ---------------------------- | --------------------------- |
| MetalLB instalado         | `kubectl apply ... metallb`  | Para simular LoadBalancer   |
| IP pool correto           | Range IPs na rede local      | IPs acessíveis no host      |
| Service tipo LoadBalancer | Para expor Envoy             | Receber tráfego externo     |
| Roteamento no host        | IP MetalLB acessível do host | Para navegação e curl       |
| Envoy Gateway configurado | Listeners e rotas corretas   | Para rotear tráfego interno |

---

Se quiser, posso ajudar a montar o YAML completo para deploy do Envoy Gateway + Service LoadBalancer + MetalLB configurado para seu ambiente kind.

Quer?
