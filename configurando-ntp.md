# Configuração de NTP com Chrony — ICP-Brasil

## Objetivo

Configurar uma máquina virtual Linux para sincronizar data e hora diretamente com os servidores NTP da ICP-Brasil, utilizando endereços IP fixos.

Essa configuração oferece redundância e evita que a sincronização dependa da resolução de nomes por DNS.

## Servidores NTP

| Servidor | Endereço IP | Prioridade |
| --- | --- | --- |
| NTP ICP-Brasil 1 | `189.9.36.90` | Preferencial |
| NTP ICP-Brasil 2 | `200.130.30.17` | Secundário |

> [!IMPORTANT]
> A configuração do Chrony é local. Ela deve ser aplicada em cada VM ou adicionada ao template antes da clonagem. Mesmo quando herdada de um template, cada VM precisa conseguir acessar os servidores NTP pela rede.

## Procedimento

### 1. Fazer backup da configuração atual

Antes de alterar o Chrony, crie uma cópia de segurança do arquivo principal:

```bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
```

### 2. Desabilitar o pool NTP padrão do Debian

Abra o arquivo principal de configuração:

```bash
sudo vim /etc/chrony/chrony.conf
```

Localize a seguinte diretiva:

```text
pool 2.debian.pool.ntp.org iburst
```

Comente a linha adicionando `#` no início:

```text
# pool 2.debian.pool.ntp.org iburst
```

Salve o arquivo e feche o editor.

> [!NOTE]
> Caso essa linha não exista, procure outras diretivas `pool` ou `server` que forneçam servidores NTP públicos e avalie se também devem ser desabilitadas.

### 3. Criar o arquivo de fontes da ICP-Brasil

Confirme que o arquivo `/etc/chrony/chrony.conf` contém a diretiva responsável por carregar as fontes adicionais:

```text
sourcedir /etc/chrony/sources.d
```

Se a diretiva não existir, adicione-a ao arquivo antes de prosseguir.

Crie um arquivo exclusivo para os servidores NTP da ICP-Brasil:

```bash
sudo vim /etc/chrony/sources.d/icpbrasil.sources
```

Adicione o conteúdo abaixo:

```text
server 189.9.36.90 iburst prefer
server 200.130.30.17 iburst
```

Salve o arquivo e feche o editor.

O parâmetro `iburst` acelera a sincronização inicial. O parâmetro `prefer` indica que o primeiro endereço deve ser priorizado quando estiver disponível e apto para sincronização.

### 4. Reiniciar o Chrony e solicitar sincronização

Reinicie o serviço e force uma nova tentativa de comunicação com as fontes configuradas:

```bash
sudo systemctl restart chrony
sudo chronyc online
sudo chronyc burst 4/4
sleep 10
```

## Validação

### 1. Verificar as fontes NTP

Execute:

```bash
chronyc sources -v
chronyc tracking
```

Na saída de `chronyc sources -v`, o resultado esperado é semelhante a:

```text
^* 189.9.36.90
^+ 200.130.30.17
```

Os principais indicadores são:

| Indicador | Significado |
| --- | --- |
| `^*` | Fonte atualmente selecionada para sincronizar o relógio. |
| `^+` | Fonte válida que está sendo combinada com a fonte selecionada. |
| `^?` | Fonte ainda indisponível ou sem resposta válida. |

O endereço marcado com `^*` pode mudar conforme disponibilidade, qualidade da conexão e precisão observada pelo Chrony. O requisito principal é que pelo menos uma das fontes válidas esteja selecionada.

### 2. Corrigir o relógio e aguardar a sincronização

Depois que uma fonte aparecer como válida, execute:

```bash
sudo chronyc makestep
chronyc waitsync 30 1.0
date -Is
```

O comando `makestep` aplica imediatamente uma correção significativa no relógio, quando necessária. O `waitsync` aguarda o Chrony atingir o limite de correção especificado, e o `date -Is` exibe a data e a hora finais no formato ISO 8601.

> [!CAUTION]
> Como `chronyc makestep` pode provocar um salto no relógio do sistema, execute-o preferencialmente antes de iniciar aplicações sensíveis a horário, bancos de dados ou serviços de cluster.

## Verificação adicional

Para confirmar o estado do serviço e a sincronização reconhecida pelo sistema operacional, execute:

```bash
systemctl is-active chrony
timedatectl show -p NTPSynchronized
```

O resultado esperado é:

```text
active
NTPSynchronized=yes
```

## Solução de problemas

Se os dois servidores permanecerem com o indicador `^?`, verifique:

1. Se a VM possui rota até `189.9.36.90` e `200.130.30.17`.
2. Se o firewall permite tráfego de saída e retorno pela porta `UDP/123`.
3. Se o serviço está ativo e sem erros:

   ```bash
   sudo systemctl status chrony
   sudo journalctl -u chrony --since "10 minutes ago"
   ```

4. Se o Chrony carregou o arquivo criado:

   ```bash
   chronyc sources -v
   ```

   Os endereços `189.9.36.90` e `200.130.30.17` devem aparecer na listagem, ainda que estejam temporariamente marcados com outro indicador.

Após corrigir qualquer falha de conectividade, repita:

```bash
sudo chronyc online
sudo chronyc burst 4/4
sleep 10
chronyc sources -v
```

## Resumo dos comandos

```bash
sudo vim /etc/chrony/chrony.conf
sudo vim /etc/chrony/sources.d/icpbrasil.sources

sudo systemctl restart chrony
sudo chronyc online
sudo chronyc burst 4/4
sleep 10

chronyc sources -v
chronyc tracking

sudo chronyc makestep
chronyc waitsync 30 1.0
date -Is
```
