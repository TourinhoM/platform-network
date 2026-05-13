# deploy-cert-manager

PKI interna do cluster k3s. Hospeda **somente infra cluster-wide**:
ClusterIssuers e a `Certificate` que gera o CA interno.

`Certificate`s de workload (keycloak-tls, monitoring-tls, etc.) **nao
ficam aqui** — ficam no repo do workload que os consome. Mesma
filosofia do Ingress: quem usa o cert e dono do cert.

## O que este repo contem

- **`selfsigned-bootstrap` ClusterIssuer** — issuer "fundacao", so
  emite o proprio CA.
- **`lab-ca` Certificate (ns cert-manager)** — gera o Secret
  `lab-ca-bundle` (CA cert + key) com 10 anos de validade.
- **`ca-issuer` ClusterIssuer** — issuer de uso, lastreado pelo
  `lab-ca-bundle`. Todos os workloads referenciam ele.

**Nao contem** o controller cert-manager — vem do Helm chart upstream
referenciado em
`deploy-argocd/apps/applications/security-cert-manager.yaml`.

## Topologia

```
selfsigned-bootstrap (ClusterIssuer)
        |
        v
lab-ca (Certificate, ns cert-manager, isCA: true, 10y)
        |
        v
Secret lab-ca-bundle (CA cert + key)
        |
        v
ca-issuer (ClusterIssuer, ca.secretName=lab-ca-bundle)
        ^
        |
        +-- referenciado por Certificates em outros repos:
            - deploy-keycloak/base/certificate-keycloak.yaml
            - deploy-stack-monitor/.../monitoring-tls.yaml (futuro)
            - deploy-dtrack/.../dtrack-tls.yaml (futuro)
            - etc.
```

cert-manager reconcilia em cascata. Argo aplica tudo deste repo de
uma vez; controller agenda emissao quando dependencia ficar Ready.

## Como adicionar TLS pra um workload novo

**Nao mexe neste repo.** No repo do workload:

1. Cria um `Certificate` no namespace do workload com
   `issuerRef.name: ca-issuer` (ClusterIssuer, cluster-scoped).
2. Define `secretName: <nome-do-secret>` — esse Secret e o que o
   Ingress vai referenciar.
3. Em `dnsNames`, lista todos os hosts daquele namespace (multi-SAN
   se cobre mais de um — ex: `grafana.local` e `prometheus.local`
   no mesmo Certificate `monitoring-tls`).

Exemplo minimo:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: minha-app-tls
spec:
  secretName: minha-app-tls
  duration: 2160h          # 90 dias
  renewBefore: 360h        # 15 dias
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  dnsNames:
    - minha-app.local
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
```

## Como verificar

```bash
# Estado dos Issuers
kubectl get clusterissuer

# Estado do CA Certificate (precisa Ready=True)
kubectl -n cert-manager get certificate lab-ca

# CA bundle pronto pra import no trust store
kubectl -n cert-manager get secret lab-ca-bundle \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > lab-ca.crt
```

## CA trust no host

Pra browsers/clientes de fora do cluster aceitarem certs sem warning,
importa o CA cert:

```bash
# Linux (Debian/Ubuntu)
sudo cp lab-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# WSL: importa lab-ca.crt em "Certificados confiaveis raiz" via
# certmgr.msc no Windows
```

## Pipeline CI

- `lint-k8s.yml` — kubeconform + kube-linter + polaris sobre `kustomize build`
- `commit-lint.yml` — conventional commits

Reusam workflows do `org-ci-platform`.
