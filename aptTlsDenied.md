# Correção do repositório APT e configuração do Chrony no Ubuntu 24.04

Este documento descreve como corrigir o acesso aos repositórios do Ubuntu 24.04 LTS (*Noble Numbat*) quando o mirror brasileiro apresenta erro de certificado TLS e, após a recuperação do APT, como instalar e configurar o Chrony para utilizar os servidores NTP do ICP-Brasil.

## Cenário do problema

O acesso ao alias brasileiro por HTTPS falhou porque o certificado apresentado não corresponde ao nome `br.archive.ubuntu.com`. Um erro típico é:

```text
curl: (60) SSL: no alternative certificate subject name matches target host name 'br.archive.ubuntu.com'
```

Isso significa que, nesse caminho de rede, não foi possível validar a identidade TLS do servidor. A causa pode estar no próprio mirror, na resolução DNS ou em algum proxy que intercepte a conexão.

> [!WARNING]
> Não utilize `curl -k`, `--insecure`, `Acquire::https::Verify-Peer=false`, `trusted=yes` ou qualquer outra opção que desative as verificações de segurança.

## Segurança do APT ao utilizar HTTP

O uso de HTTP neste procedimento é apenas uma alternativa para o mirror que não apresenta um certificado TLS válido. Embora o transporte não seja criptografado, o APT continua verificando a autenticidade do conteúdo por meio do mecanismo `apt-secure`:

- o arquivo `InRelease` é autenticado por assinatura OpenPGP;
- os índices do repositório são validados pelos hashes registrados nos metadados assinados;
- os pacotes baixados são validados pelos hashes presentes nesses índices.

Portanto, não se deve aceitar repositórios sem assinatura nem ignorar erros de autenticação. Sempre que possível, deve-se preferir um mirror HTTPS com certificado válido.

## Pré-requisitos

- Ubuntu 24.04 LTS (*Noble Numbat*);
- acesso administrativo com `sudo`;
- conectividade IPv4 com a Internet;
- acesso TCP às portas `80` e `443` para os mirrors;
- acesso UDP à porta `123` para os servidores NTP.

O arquivo de repositórios utilizado neste guia é:

```text
/etc/apt/sources.list.d/ubuntu.sources
```

Antes de alterá-lo, confirme a configuração atual:

```bash
grep -nE '^(URIs|Suites|Components):' \
  /etc/apt/sources.list.d/ubuntu.sources
```

Crie também uma cópia de segurança:

```bash
sudo cp -a \
  /etc/apt/sources.list.d/ubuntu.sources \
  "/etc/apt/sources.list.d/ubuntu.sources.bak.$(date +%Y%m%d-%H%M%S)"
```

## Fluxo de decisão

| Teste | Resultado | Ação |
| --- | --- | --- |
| Mirror brasileiro por HTTP | Status `200` | Configurar `http://br.archive.ubuntu.com/ubuntu/` |
| Mirror brasileiro por HTTP | Falha | Testar o mirror C3SL/UFPR por HTTPS |
| Mirror C3SL/UFPR por HTTPS | Status `200` | Configurar `https://ubuntu.c3sl.ufpr.br/ubuntu/` |
| Ambos os mirrors | Falha | Interromper o procedimento e diagnosticar DNS, proxy, firewall ou rota |

> [!IMPORTANT]
> Só instale o Chrony depois que `sudo apt update` terminar sem linhas `Err:` e sem o aviso `W: Failed to fetch`.

## Opção 1 — Utilizar o mirror brasileiro por HTTP

### 1. Testar o arquivo `InRelease`

Execute:

```bash
curl -4 -L -I --max-time 20 \
  http://br.archive.ubuntu.com/ubuntu/dists/noble/InRelease
```

O teste é considerado bem-sucedido quando a resposta final contém um status `200`, por exemplo:

```text
HTTP/1.1 200 OK
```

Dependendo do servidor, a resposta também pode aparecer como `HTTP/2 200`.

### 2. Alterar a URL do repositório

Se o teste retornar `200`, substitua HTTPS por HTTP:

```bash
sudo sed -i \
  's|https://br.archive.ubuntu.com/ubuntu/|http://br.archive.ubuntu.com/ubuntu/|g' \
  /etc/apt/sources.list.d/ubuntu.sources
```

Confirme a alteração:

```bash
grep -n '^URIs:' /etc/apt/sources.list.d/ubuntu.sources
```

### 3. Atualizar os índices do APT

```bash
sudo apt clean
sudo apt update
```

Se a atualização terminar sem `Err:` e sem `W: Failed to fetch`, instale o Chrony:

```bash
sudo apt install -y chrony
```

Depois da instalação, avance para [Configuração dos servidores NTP](#configuração-dos-servidores-ntp).

## Opção 2 — Utilizar o mirror C3SL/UFPR por HTTPS

Use esta opção somente se o mirror brasileiro por HTTP também falhar.

### 1. Testar o mirror alternativo

```bash
curl -4 -L -I --max-time 20 \
  https://ubuntu.c3sl.ufpr.br/ubuntu/dists/noble/InRelease
```

Prossiga apenas se a resposta final apresentar status `200`.

### 2. Configurar o mirror alternativo

O comando abaixo substitui o alias brasileiro independentemente de ele estar configurado com HTTP ou HTTPS:

```bash
sudo sed -i -E \
  's|https?://br\.archive\.ubuntu\.com/ubuntu/|https://ubuntu.c3sl.ufpr.br/ubuntu/|g' \
  /etc/apt/sources.list.d/ubuntu.sources
```

Confirme a nova URL:

```bash
grep -n '^URIs:' /etc/apt/sources.list.d/ubuntu.sources
```

### 3. Atualizar o APT e instalar o Chrony

```bash
sudo apt clean
sudo apt update
```

Se a atualização terminar sem erros de download, execute:

```bash
sudo apt install -y chrony
```

> [!CAUTION]
> Se os dois mirrors falharem, não prossiga com a instalação. Verifique a resolução DNS, as regras de firewall, a existência de proxy e a rota de saída da máquina.

## Configuração dos servidores NTP

### 1. Criar a fonte NTP do ICP-Brasil

Crie um arquivo dedicado para facilitar a manutenção da configuração:

```bash
sudo tee /etc/chrony/sources.d/icpbrasil.sources >/dev/null <<'EOF'
server 200.130.30.17 iburst prefer
server 189.9.36.90 iburst
EOF
```

Significado das opções:

- `iburst`: acelera a sincronização inicial enviando uma sequência curta de consultas;
- `prefer`: indica que o servidor `200.130.30.17` deve ser priorizado quando estiver disponível e saudável.

Confirme se o arquivo principal do Chrony carrega o diretório de fontes:

```bash
grep -nE '^[[:space:]]*sourcedir[[:space:]]+/etc/chrony/sources\.d' \
  /etc/chrony/chrony.conf
```

O Ubuntu normalmente já inclui essa diretiva. Caso nenhum resultado seja exibido, adicione-a:

```bash
echo 'sourcedir /etc/chrony/sources.d' | \
  sudo tee -a /etc/chrony/chrony.conf >/dev/null
```

### 2. Desativar os pools padrão

Comente as diretivas `pool` e `server` existentes no arquivo principal:

```bash
sudo sed -i -E \
  's/^([[:space:]]*(pool|server)[[:space:]])/# \1/' \
  /etc/chrony/chrony.conf
```

Esse comando não altera o arquivo `/etc/chrony/sources.d/icpbrasil.sources` criado anteriormente.

> [!NOTE]
> Fontes NTP recebidas por DHCP podem continuar aparecendo no `chronyc sources -v`. O comando acima desativa apenas as diretivas estáticas `pool` e `server` presentes em `/etc/chrony/chrony.conf`.

### 3. Validar a sintaxe antes de reiniciar

```bash
sudo chronyd -p
```

O comando deve imprimir a configuração processada sem apresentar erro de sintaxe.

## Ativação e sincronização

Habilite o serviço, recarregue a configuração e solicite a sincronização:

```bash
sudo systemctl enable --now chrony
sudo systemctl restart chrony
sudo chronyc online
sudo chronyc burst 4/4
sleep 10
sudo chronyc makestep
chronyc waitsync 30 1.0
```

O comando `makestep` permite corrigir imediatamente uma diferença significativa no relógio. Já `waitsync 30 1.0` aguarda a sincronização por até 30 tentativas, exigindo uma correção restante inferior a 1 segundo.

> [!WARNING]
> O `makestep` pode avançar ou retroceder o relógio de forma imediata. Execute-o preferencialmente antes de iniciar aplicações sensíveis a alterações bruscas de horário, como bancos de dados, sistemas distribuídos e serviços de autenticação.

## Validação

Execute:

```bash
chronyc sources -v
chronyc tracking
timedatectl status
date -Is
```

### Resultado esperado em `chronyc sources -v`

Um dos servidores deve aparecer com `^*`, indicando que foi selecionado como fonte atual:

```text
^* 200.130.30.17
^+ 189.9.36.90
```

Os símbolos mais relevantes são:

| Símbolo | Significado |
| --- | --- |
| `^*` | Servidor selecionado para sincronizar o relógio |
| `^+` | Servidor válido combinado com a fonte selecionada |
| `^-` | Servidor válido que não está sendo usado no cálculo atual |
| `^?` | Servidor sem resposta, inacessível ou ainda sem amostras suficientes |
| `^x` | Fonte considerada inconsistente pelo Chrony |

Não é obrigatório que os dois servidores estejam com `^*`; apenas uma fonte pode ser selecionada por vez.

### Resultado esperado em `chronyc tracking`

Verifique principalmente:

- `Reference ID`: deve corresponder a uma fonte NTP válida;
- `Stratum`: deve ser maior que zero;
- `Leap status`: deve aparecer como `Normal`.

### Resultado esperado em `timedatectl status`

```text
System clock synchronized: yes
NTP service: active
```

O comando `date -Is` deve exibir a data, a hora e o fuso horário corretos no formato ISO 8601.

## Solução de problemas

### O APT ainda apresenta falha

Verifique novamente as URLs configuradas:

```bash
grep -nE '^(URIs|Suites):' /etc/apt/sources.list.d/ubuntu.sources
```

Teste DNS e conectividade:

```bash
getent ahostsv4 br.archive.ubuntu.com
getent ahostsv4 ubuntu.c3sl.ufpr.br
curl -4 -L -I --max-time 20 \
  http://br.archive.ubuntu.com/ubuntu/dists/noble/InRelease
curl -4 -L -I --max-time 20 \
  https://ubuntu.c3sl.ufpr.br/ubuntu/dists/noble/InRelease
```

Não ignore erros como:

- `The following signatures couldn't be verified`;
- `The repository ... is not signed`;
- `W: Failed to fetch`;
- `Err:` durante o download dos índices.

### O Chrony não sincroniza

Consulte o estado do serviço e os logs:

```bash
systemctl status chrony --no-pager
journalctl -u chrony -n 100 --no-pager
chronyc activity
chronyc sources -v
chronyc sourcestats -v
```

Se os servidores permanecerem com `^?`, verifique se o tráfego UDP na porta `123` está liberado entre a máquina e os endereços:

```text
200.130.30.17:123/UDP
189.9.36.90:123/UDP
```

## Reversão da configuração do APT

Para restaurar o arquivo original, localize a cópia criada no início:

```bash
ls -1t /etc/apt/sources.list.d/ubuntu.sources.bak.*
```

Em seguida, substitua `<ARQUIVO_DE_BACKUP>` pelo caminho escolhido:

```bash
sudo cp -a \
  <ARQUIVO_DE_BACKUP> \
  /etc/apt/sources.list.d/ubuntu.sources

sudo apt clean
sudo apt update
```

## Checklist final

- [ ] O mirror selecionado respondeu com status HTTP `200`.
- [ ] `sudo apt update` terminou sem `Err:` e sem `W: Failed to fetch`.
- [ ] O pacote `chrony` foi instalado.
- [ ] O serviço `chrony` está ativo e habilitado.
- [ ] Um dos servidores ICP-Brasil aparece com `^*`.
- [ ] `chronyc tracking` apresenta `Leap status: Normal`.
- [ ] `timedatectl status` apresenta `System clock synchronized: yes`.
- [ ] `date -Is` exibe a data, a hora e o fuso horário corretos.
