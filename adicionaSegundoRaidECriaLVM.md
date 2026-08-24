# Configuração de um segundo RAID 1 com LVM-thin no Proxmox

Este guia mostra como identificar uma controladora MegaRAID, instalar as ferramentas de gerenciamento, criar um segundo RAID 1 com dois discos e disponibilizá-lo no Proxmox como o armazenamento `local-lvm-2`.

## Cenário utilizado

Os comandos deste documento consideram o seguinte ambiente:

| Item | Valor |
| --- | --- |
| Controladora | `/c0` |
| Enclosure | `64` |
| Discos do RAID existente | `64:0` e `64:1` |
| Novos discos | `64:2` e `64:3` |
| Novo disco apresentado ao Linux | `/dev/sdb` |
| Nome do novo RAID | `RAID1_DADOS` |
| Volume Group | `vg_raid1_dados` |
| Thin pool | `data` |
| Storage no Proxmox | `local-lvm-2` |
| Nó do Proxmox | `lab-cgope-01` |

> [!CAUTION]
> Alguns comandos deste guia removem configurações RAID e assinaturas de dados. Confirme os identificadores dos discos em seu servidor antes de executá-los. Nunca suponha que os discos serão sempre `64:2`, `64:3` ou `/dev/sdb`.

> [!IMPORTANT]
> Execute os comandos como `root`. Execute um bloco por vez, confira o resultado e pare imediatamente se aparecer algum erro ou resultado diferente do esperado.

## 1. Identificar a controladora e os utilitários disponíveis

Pesquise quais ferramentas relacionadas à controladora estão disponíveis nos repositórios configurados:

```bash
apt-cache search --names-only 'storcli|megacli|megactl'
```

Identifique também a controladora RAID, SAS ou SCSI presente no servidor:

```bash
lspci -nnk | grep -A4 -Ei 'RAID|SAS|SCSI'
```

Essas consultas não alteram a configuração do servidor.

## 2. Consultar a controladora com o `megasasctl`

Se o pacote `megactl` estiver disponível, instale-o:

```bash
apt install -y megactl
```

Liste as controladoras, os volumes RAID e os discos físicos:

```bash
megasasctl -v
```

O comando acima é apenas uma consulta e não altera o RAID.

Para executar uma verificação resumida de saúde:

```bash
megasasctl -H
```

> [!NOTE]
> Quando todos os dispositivos estão saudáveis, `megasasctl -H` pode terminar sem imprimir nada. Use `megasasctl -v` como consulta principal.

Para consultar o manual instalado no sistema:

```bash
man megasasctl
```

## 3. Instalar o StorCLI da Broadcom

O StorCLI permite consultar e administrar a controladora MegaRAID.

### 3.1. Instalar as dependências

```bash
apt install -y curl unzip file
```

### 3.2. Baixar o pacote oficial

Crie um diretório temporário e faça o download diretamente do domínio da Broadcom:

```bash
STORCLI_DIR=$(mktemp -d /tmp/storcli.XXXXXX)
cd "$STORCLI_DIR"

curl -fL --retry 3 --connect-timeout 20 \
  -o storcli.zip \
  'https://docs.broadcom.com/docs-and-downloads/007.2705.0000.0000_storcli_rel.zip'
```

### 3.3. Validar o arquivo baixado

Confirme que o arquivo é realmente um ZIP, verifique seu tamanho e teste sua integridade:

```bash
file storcli.zip
stat -c 'Tamanho: %s bytes' storcli.zip
unzip -t storcli.zip | tail -n 2
```

Para o pacote usado neste guia, o tamanho de referência é aproximadamente `34.699.985 bytes`. O teste de integridade deve terminar com:

```text
No errors detected
```

> [!WARNING]
> Não prossiga se o arquivo não for identificado como ZIP ou se o teste apresentar erros. O tamanho pode mudar caso a Broadcom substitua o pacote, portanto a validação com `unzip -t` é a confirmação principal.

### 3.4. Extrair e instalar

```bash
unzip -q storcli.zip

unzip -q \
  storcli_rel/Unified_storcli_all_os.zip \
  -d unified

dpkg -i \
  unified/Unified_storcli_all_os/Ubuntu/storcli_007.2705.0000.0000_all.deb
```

### 3.5. Testar a instalação

Os comandos abaixo apenas mostram a versão e consultam a controladora:

```bash
/opt/MegaRAID/storcli/storcli64 -v
/opt/MegaRAID/storcli/storcli64 /c0 show
```

## 4. Verificar a saúde dos novos discos

Consulte erros, alertas S.M.A.R.T. e temperatura dos discos `64:2` e `64:3`:

```bash
/opt/MegaRAID/storcli/storcli64 /c0/e64/s2 show all \
  | grep -Ei 'Media Error|Other Error|Predictive Failure|S.M.A.R.T|Drive Temperature'

/opt/MegaRAID/storcli/storcli64 /c0/e64/s3 show all \
  | grep -Ei 'Media Error|Other Error|Predictive Failure|S.M.A.R.T|Drive Temperature'
```

Antes de continuar, confirme que:

- os discos `64:2` e `64:3` são realmente os discos novos;
- não existem erros de mídia ou falhas preditivas;
- nenhuma informação importante depende da configuração estrangeira desses discos.

## 5. Remover a configuração estrangeira

Defina uma variável para facilitar os próximos comandos:

```bash
STORCLI=/opt/MegaRAID/storcli/storcli64
```

Salve o estado atual da controladora e os detalhes da configuração estrangeira:

```bash
$STORCLI /c0 show all > /root/storcli-before-raid2.txt
$STORCLI /c0/fall show all > /root/storcli-foreign-before-clear.txt
```

Revise a configuração antes de removê-la:

```bash
$STORCLI /c0/fall show all
```

> [!CAUTION]
> O próximo comando exclui todas as configurações estrangeiras detectadas na controladora `/c0`. Continue somente se elas pertencerem aos discos novos `64:2` e `64:3` e puderem ser descartadas.

Remova a configuração estrangeira:

```bash
$STORCLI /c0/fall delete
```

Confira novamente todos os discos físicos:

```bash
$STORCLI /c0/eall/sall show
```

O resultado esperado é semelhante a:

```text
64:0  Onln   0
64:1  Onln   0
64:2  UGood  -
64:3  UGood  -
```

Interpretação:

- `Onln`: disco on-line e pertencente a um RAID existente;
- `UGood`: disco saudável, não configurado e disponível para uso;
- `0`: Disk Group do primeiro RAID;
- `-`: o disco ainda não pertence a um Disk Group.

Não continue se `64:0` ou `64:1` deixarem de aparecer como `Onln`, ou se os discos novos não aparecerem como `UGood`.

## 6. Criar o segundo RAID 1

Crie o RAID 1 usando exclusivamente os discos `64:2` e `64:3`:

```bash
$STORCLI /c0 add vd \
  type=raid1 \
  name=RAID1_DADOS \
  drives=64:2-3 \
  wt nora direct
```

Parâmetros principais:

| Parâmetro | Função |
| --- | --- |
| `type=raid1` | Cria um espelhamento RAID 1 |
| `name=RAID1_DADOS` | Define o nome do volume virtual |
| `drives=64:2-3` | Usa somente os discos `64:2` e `64:3` |
| `wt` | Define a política de escrita como *Write Through* |
| `nora` | Desativa o *Read Ahead* |
| `direct` | Define a política de cache de E/S como *Direct I/O* |

Verifique o volume virtual, os discos físicos e os dispositivos apresentados ao Linux:

```bash
$STORCLI /c0/vall show
$STORCLI /c0/eall/sall show
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

O resultado esperado deve indicar:

```text
DG/VD 1/1  RAID1  Optl  aproximadamente 557.861 GB
64:2       Onln   1
64:3       Onln   1
sdb        aproximadamente 558G
```

O estado `Optl` informa que o RAID está em condição **Optimal**.

## 7. Limpar metadados antigos do novo volume

O novo `/dev/sdb` pode continuar exibindo partições e assinaturas LVM herdadas da configuração estrangeira.

Confira novamente o dispositivo antes de apagar qualquer assinatura:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL /dev/sdb
wipefs --no-act /dev/sdb
```

> [!CAUTION]
> Os próximos comandos apagam as assinaturas de sistemas de arquivos, partições e LVM existentes em `/dev/sdb`. Eles são irreversíveis para fins práticos. Execute-os somente depois de confirmar que `/dev/sdb` é o novo RAID e que seus dados podem ser descartados.

Limpe primeiro as partições antigas e depois o disco:

```bash
wipefs --all --force /dev/sdb1
wipefs --all --force /dev/sdb2
wipefs --all --force /dev/sdb3
wipefs --all --force /dev/sdb
```

Atualize a tabela de partições reconhecida pelo kernel:

```bash
blockdev --rereadpt /dev/sdb
udevadm settle
```

Confira o resultado:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL /dev/sdb
wipefs --no-act /dev/sdb
```

O esperado é que não existam partições ou assinaturas antigas:

```text
sdb  557.9G  disk  ServeRAID M1115
```

## 8. Criar o LVM-thin

O armazenamento `local-lvm-2` será criado exclusivamente em `/dev/sdb`.

> [!IMPORTANT]
> Execute um comando por vez. Se algum comando falhar, pare e investigue antes de continuar.

### 8.1. Criar o Physical Volume e o Volume Group

```bash
pvcreate /dev/sdb
vgcreate vg_raid1_dados /dev/sdb
```

### 8.2. Criar o thin pool

Use 95% do espaço livre do Volume Group:

```bash
lvcreate \
  --type thin-pool \
  --name data \
  --extents 95%FREE \
  vg_raid1_dados
```

Os 5% restantes ficam livres no Volume Group para uma possível expansão dos metadados ou reparo do thin pool.

## 9. Adicionar o armazenamento ao Proxmox

Registre o thin pool como `local-lvm-2`, limitado ao nó `lab-cgope-01`:

```bash
pvesm add lvmthin local-lvm-2 \
  --vgname vg_raid1_dados \
  --thinpool data \
  --content images,rootdir \
  --nodes lab-cgope-01
```

O conteúdo permitido será:

- `images`: discos de máquinas virtuais;
- `rootdir`: volumes de containers LXC.

> [!NOTE]
> O LVM-thin criado em um disco local não é um armazenamento compartilhado entre os nós do cluster. Por isso, o storage foi restringido ao `lab-cgope-01`.

## 10. Validar a configuração final

Verifique o Physical Volume, o Volume Group, o thin pool e os storages do Proxmox:

```bash
pvs
vgs
lvs -a -o lv_name,vg_name,lv_size,segtype,data_percent,metadata_percent
pvesm status
```

Confirme que:

- `/dev/sdb` pertence ao `vg_raid1_dados`;
- o thin pool `data` está ativo;
- `local-lvm-2` aparece como `active` no `pvesm status`;
- os percentuais `Data%` e `Meta%` não indicam uso inesperado;
- os dois discos do novo RAID permanecem `Onln`;
- o novo volume virtual permanece em estado `Optl`.

## 11. Salvar o estado final da controladora

Gere um relatório final para comparação futura:

```bash
/opt/MegaRAID/storcli/storcli64 /c0 show all \
  > /root/storcli-after-raid2.txt
```

Os relatórios salvos serão:

| Arquivo | Conteúdo |
| --- | --- |
| `/root/storcli-before-raid2.txt` | Estado da controladora antes da criação do RAID |
| `/root/storcli-foreign-before-clear.txt` | Configuração estrangeira antes da remoção |
| `/root/storcli-after-raid2.txt` | Estado final da controladora |

