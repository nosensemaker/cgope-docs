# Configuração de DNS para a rede 192.168.7.0/24

## Objetivo

Configurar os servidores localizados na rede `192.168.7.0/24` para utilizarem servidores DNS acessíveis a partir desse segmento de rede.

## Contexto

O firewall bloqueia a comunicação entre a rede `192.168.7.0/24` e a rede `192.168.16.0/24`. Por esse motivo, o servidor DNS `192.168.16.1` não pode ser utilizado pelos equipamentos da rede 7.

Para esses equipamentos, devem ser configurados os seguintes servidores DNS:

| Prioridade | Servidor DNS |
| --- | --- |
| Primário | `192.168.6.38` |
| Secundário | `192.168.6.39` |

## Procedimento

> É necessário executar o procedimento com o usuário `root` ou com privilégios administrativos.

### 1. Abrir o arquivo de configuração

```bash
nano /etc/resolv.conf
```

### 2. Configurar os servidores DNS

Remova ou substitua as entradas `nameserver` existentes e mantenha o arquivo com as seguintes configurações:

```conf
nameserver 192.168.6.38
nameserver 192.168.6.39
```

Caso o arquivo já possua uma diretiva `search`, ela pode ser mantida. Exemplo:

```conf
search cluster.cgope01
nameserver 192.168.6.38
nameserver 192.168.6.39
```

### 3. Salvar e sair do Nano

No editor Nano, pressione as teclas abaixo nesta ordem:

1. `Ctrl + O` — salva as alterações;
2. `Enter` — confirma o nome do arquivo;
3. `Ctrl + X` — fecha o editor.

## Validação

Confira se o arquivo foi salvo corretamente:

```bash
cat /etc/resolv.conf
```

Em seguida, teste a resolução de nomes:

```bash
getent hosts deb.debian.org
```

Se o comando retornar um ou mais endereços IP, a resolução DNS está funcionando.

Também é possível testar a conectividade com cada servidor DNS:

```bash
ping -c 3 192.168.6.38
ping -c 3 192.168.6.39
```

## Observação

Em alguns ambientes, o arquivo `/etc/resolv.conf` é gerenciado automaticamente e pode ser sobrescrito após uma reinicialização ou alteração da rede. Caso isso ocorra, configure os mesmos endereços DNS de forma permanente no gerenciador de rede utilizado pelo servidor ou na interface administrativa do Proxmox.
