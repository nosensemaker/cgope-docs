# Instalação do Docker Engine no Ubuntu

Este manual apresenta uma instalação simples do Docker Engine usando o repositório oficial do Docker.

## 1. Instalar as dependências

Atualize o índice de pacotes e instale os pacotes necessários:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

## 2. Adicionar a chave oficial do Docker

Crie o diretório de chaves do APT e baixe a chave oficial:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

A chave permite que o APT verifique a autenticidade dos pacotes fornecidos pelo Docker.

## 3. Adicionar o repositório oficial

Copie e execute o bloco completo abaixo:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Atualize novamente o índice de pacotes:

```bash
sudo apt update
```

## 4. Instalar o Docker Engine

Instale o Docker Engine, o cliente, o containerd, o Buildx e o Docker Compose:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## 5. Verificar o serviço

Confirme se o Docker está em execução:

```bash
sudo systemctl status docker --no-pager
```

O estado esperado é `active (running)`. Caso o serviço não esteja ativo, execute:

```bash
sudo systemctl enable --now docker
```

## 6. Testar a instalação

Execute o contêiner de teste:

```bash
sudo docker run hello-world
```

Se uma mensagem de boas-vindas for exibida, o Docker foi instalado corretamente.

Também é possível verificar as versões instaladas:

```bash
docker --version
docker compose version
```

## 7. Usar o Docker sem `sudo` (opcional)

Por padrão, apenas o usuário `root` pode acessar o socket do Docker. Para permitir que o usuário atual execute comandos sem `sudo`, adicione-o ao grupo `docker`:

```bash
getent group docker > /dev/null || sudo groupadd docker
sudo usermod -aG docker "$USER"
```

Ative a nova associação de grupo na sessão atual:

```bash
newgrp docker
```

Teste novamente, agora sem `sudo`:

```bash
docker run hello-world
```
