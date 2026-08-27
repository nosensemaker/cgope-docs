# Correção do erro `SysctlForbidden` no Kubernetes

Este documento descreve como corrigir a falha de inicialização de um pod causada pelo uso do sysctl inseguro:

```text
net.ipv6.conf.all.disable_ipv6
```

O exemplo considera o pod `api-validar-0`, no namespace `validar`.

## Sumário

- [Sintoma](#sintoma)
- [Causa](#causa)
- [Correção recomendada](#correção-recomendada)
- [Alternativa: liberar o sysctl no kubelet](#alternativa-liberar-o-sysctl-no-kubelet)
- [Validação](#validação)
- [Rollback](#rollback)
- [Solução de problemas](#solução-de-problemas)
- [Referências](#referências)

## Sintoma

O pod é atribuído a um worker, mas não consegue iniciar. Nos eventos aparece uma mensagem semelhante a:

```text
Warning  SysctlForbidden  pod/api-validar-0
forbidden sysctl: "net.ipv6.conf.all.disable_ipv6" not allowlisted
```

Confira o pod e seus eventos:

```bash
kubectl get pod api-validar-0 -n validar -o wide

kubectl describe pod api-validar-0 -n validar

kubectl get events -n validar \
  --field-selector involvedObject.name=api-validar-0 \
  --sort-by=.lastTimestamp
```

Para consultar os sysctls solicitados pelo pod:

```bash
kubectl get pod api-validar-0 -n validar \
  -o jsonpath='{.spec.securityContext.sysctls}{"\n"}'
```

> **Atenção:** `net.ipv6.conf.all.disable_ipv6` e
> `net.ipv6.conf.default.disable_ipv6` são parâmetros diferentes. Utilize
> exatamente o nome exibido nos eventos do pod.

## Causa

O Kubernetes classifica os sysctls como **seguros** ou **inseguros**. Os sysctls inseguros ficam bloqueados por padrão e somente podem ser permitidos explicitamente pelo administrador em cada nó.

Por isso, o scheduler consegue atribuir o pod ao worker, mas o kubelet do nó impede sua inicialização.

Neste caso, a configuração costuma estar no `securityContext` do pod gerado pelo Helm:

```yaml
securityContext:
  sysctls:
    - name: net.ipv6.conf.all.disable_ipv6
      value: "1"
```

## Correção recomendada

A solução preferencial é remover o sysctl do chart Helm ou do manifesto Kubernetes. Isso evita alterar a política de segurança dos workers.

### 1. Localizar a configuração

No diretório do projeto, utilize o `rg`:

```bash
rg -n --hidden -g '!.git' \
  'net\.ipv6\.conf\.all\.disable_ipv6' .
```

Se o `rg` não estiver instalado, utilize:

```bash
grep -RniF --exclude-dir=.git \
  'net.ipv6.conf.all.disable_ipv6' .
```

Se a configuração for produzida por um template Helm, também confira o manifesto renderizado:

```bash
helm template <NOME_DA_RELEASE> . -f <ARQUIVO_VALUES> \
  | grep -nF -B 5 -A 5 'net.ipv6.conf.all.disable_ipv6'
```

Exemplo:

```bash
helm template api-validar . -f values-dev.yaml \
  | grep -nF -B 5 -A 5 'net.ipv6.conf.all.disable_ipv6'
```

### 2. Remover o sysctl

Remova do template ou dos valores do chart o trecho equivalente a:

```yaml
securityContext:
  sysctls:
    - name: net.ipv6.conf.all.disable_ipv6
      value: "1"
```

Não edite o pod diretamente: pods controlados por `Deployment` ou `StatefulSet` são recriados a partir do manifesto original. Em um ambiente GitOps, a alteração deve ser feita no repositório acompanhado pelo Argo CD.

### 3. Fazer a aplicação Java usar IPv4

Se a intenção era somente fazer a aplicação Java abrir sockets IPv4, configure a JVM em vez de desabilitar o IPv6 por sysctl:

```yaml
spec:
  template:
    spec:
      containers:
        - name: <NOME_DO_CONTAINER>
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-Djava.net.preferIPv4Stack=true"
```

Essa propriedade faz a JVM utilizar apenas sockets IPv4. Ela não desabilita o IPv6 no sistema operacional nem afeta outros processos do pod.

> Se `JAVA_TOOL_OPTIONS` já existir, preserve as opções atuais e acrescente
> `-Djava.net.preferIPv4Stack=true` ao mesmo valor. Não crie duas variáveis com
> o mesmo nome.

Também verifique se a aplicação não precisa acessar serviços disponíveis exclusivamente por IPv6, pois eles deixarão de ser acessíveis pela JVM com essa opção.

### 4. Publicar e sincronizar pelo Argo CD

Revise, faça o commit e envie a alteração ao repositório Git:

```bash
git diff
git add <ARQUIVOS_ALTERADOS>
git commit -m "fix: remove unsafe IPv6 sysctl"
git push
```

Sincronize a aplicação pela interface do Argo CD ou pela CLI:

```bash
argocd app sync <NOME_DA_APLICACAO>

argocd app wait <NOME_DA_APLICACAO> \
  --sync \
  --health \
  --timeout 300
```

## Alternativa: liberar o sysctl no kubelet

Use esta opção somente se o sysctl for realmente obrigatório. Sysctls inseguros podem afetar a estabilidade e a segurança do nó.

A liberação é individual por worker. Portanto, configure todos os nós nos quais o pod pode ser agendado ou restrinja o workload somente aos nós preparados, usando afinidade de nó e, preferencialmente, taints e tolerations.

> O procedimento abaixo considera um cluster criado com `kubeadm`, cujo arquivo
> normalmente é `/var/lib/kubelet/config.yaml`.

### 1. Confirmar o arquivo usado pelo kubelet

Em cada worker que receberá a configuração, execute:

```bash
sudo systemctl cat kubelet | grep -- '--config'
```

Confirme que o serviço aponta para:

```text
/var/lib/kubelet/config.yaml
```

### 2. Fazer backup da configuração

```bash
sudo cp --preserve=all \
  /var/lib/kubelet/config.yaml \
  /var/lib/kubelet/config.yaml.bak
```

### 3. Editar a configuração do kubelet

```bash
sudo vim /var/lib/kubelet/config.yaml
```

Adicione `allowedUnsafeSysctls` no nível principal do YAML:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

allowedUnsafeSysctls:
  - net.ipv6.conf.all.disable_ipv6
```

As linhas `apiVersion` e `kind` já devem existir no arquivo e foram incluídas acima apenas para mostrar o nível correto da chave.

Se `allowedUnsafeSysctls` já existir, apenas acrescente o item:

```yaml
allowedUnsafeSysctls:
  - outro.sysctl.ja.configurado
  - net.ipv6.conf.all.disable_ipv6
```

Não crie uma segunda chave `allowedUnsafeSysctls`.

### 4. Reiniciar e verificar o kubelet

Execute em um worker por vez:

```bash
sudo systemctl restart kubelet

sudo systemctl status kubelet --no-pager
```

No control plane, aguarde o nó voltar ao estado `Ready`:

```bash
kubectl wait \
  --for=condition=Ready \
  node/worker-k8s-dev-02 \
  --timeout=120s
```

Repita o procedimento nos demais workers em que o pod puder executar.

### 5. Recriar o pod

Primeiro, confirme que o pod possui um controlador responsável por recriá-lo:

```bash
kubectl get pod api-validar-0 -n validar \
  -o jsonpath='{.metadata.ownerReferences[0].kind}{"/"}{.metadata.ownerReferences[0].name}{"\n"}'
```

Se o comando retornar um controlador, como `StatefulSet/api-validar`, recrie o pod:

```bash
kubectl delete pod api-validar-0 -n validar

kubectl get pod api-validar-0 -n validar -w
```

Se não houver um controlador, não exclua o pod antes de garantir que exista um manifesto capaz de recriá-lo.

## Validação

Verifique se o novo pod está em execução e em qual nó foi iniciado:

```bash
kubectl get pod api-validar-0 -n validar -o wide
```

Confira os eventos recentes:

```bash
kubectl describe pod api-validar-0 -n validar
```

O evento `SysctlForbidden` não deve aparecer novamente. Em seguida, valide os logs da aplicação:

```bash
kubectl logs api-validar-0 -n validar --all-containers --tail=100
```

Para acompanhar a implantação completa:

```bash
kubectl get pods -n validar -w
```

## Rollback

Se a alteração no kubelet causar falha, restaure o backup no worker afetado:

```bash
sudo cp --preserve=all \
  /var/lib/kubelet/config.yaml.bak \
  /var/lib/kubelet/config.yaml

sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

No control plane, confirme novamente o estado do nó:

```bash
kubectl get node worker-k8s-dev-02
```

## Solução de problemas

### O kubelet não inicia após a alteração

Consulte os logs:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

As causas mais comuns são indentação incorreta no YAML, chave duplicada ou edição de uma seção errada. Restaure o backup se necessário.

### O erro `SysctlForbidden` continua

Verifique:

- se o pod foi recriado depois da alteração;
- em qual worker o novo pod foi agendado;
- se esse worker recebeu a mesma configuração;
- se o kubelet foi reiniciado com sucesso;
- se o serviço realmente usa `/var/lib/kubelet/config.yaml`;
- se o nome configurado é exatamente o mesmo do evento, especialmente `all` ou `default`.

### O pod utiliza `hostNetwork: true`

Sysctls `net.*` não são permitidos para pods que compartilham a rede do host. Nesse caso, remova o sysctl do pod e reavalie a necessidade da configuração.

## Referências

- [Kubernetes — Using sysctls in a Kubernetes Cluster](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Kubernetes — Kubelet Configuration (v1beta1)](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Oracle Java — Networking Properties](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/net/doc-files/net-properties.html)
- [Argo CD — `argocd app sync`](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_app_sync/)
- [Argo CD — `argocd app wait`](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_app_wait/)
