# GLPI em Docker 
<img width="350" height="150" alt="image" src="https://github.com/user-attachments/assets/f7946c92-0212-408a-9714-385139bd5d2c" />


Implementação de um sistema de gestão de ativos e chamados em uma VM Ubuntu com recursos limitados, compartilhada com um servidor RustDesk já instalado em Docker para futura integração entre o GLPI e RustDesk.

O projeto documenta a preparação da memória virtual, a instalação do GLPI com MariaDB, o acesso administrativo por túnel SSH e o procedimento futuro para publicação com domínio e HTTPS.


## Sumário

- [Ambiente e decisões](#ambiente-e-decisões)
- [1. Diagnóstico inicial](#1-diagnóstico-inicial)
- [2. Criar swap de 4 GiB](#2-criar-swap-de-4-gib)
- [3. Preparar o projeto GLPI](#3-preparar-o-projeto-glpi)
- [4. Criar o Docker Compose](#4-criar-o-docker-compose)
- [5. Iniciar e validar](#5-iniciar-e-validar)
- [6. Acessar pelo PuTTY](#6-acessar-pelo-putty)
- [7. Configurar domínio e HTTPS](#7-configurar-domínio-e-https)
- [8. Operação, backup e atualização](#8-operação-backup-e-atualização)
- [Referências oficiais](#referências-oficiais)

## Ambiente e decisões

| Componente | Ambiente documentado |
|---|---|
| Plataforma | Oracle Cloud Infrastructure — VM utilizada no Free Tier |
| Sistema | Ubuntu 22.04.5 LTS |
| Processamento | 1 núcleo AMD |
| Memória | 1 GB nominal; aproximadamente 956 MiB disponíveis ao sistema |
| Armazenamento | 200 GB provisionados; raiz ext4 de aproximadamente 194 GB |
| Serviço existente | RustDesk em Docker, containers `hbbs` e `hbbr` |
| Docker / Compose observados | 29.1.3 / v2.35.1 |
| GLPI instalado | Imagem oficial `glpi/glpi:11.0.12` |
| Banco | `mariadb:10.11` |
| Swap adicionada | Arquivo de 4 GiB |


O Docker já estava instalado. O projeto não reinstala o Docker, não recria o RustDesk e não ensina a instalação inicial desse serviço. Seus volumes, portas e política de reinício devem ser preservados.

A swap oferece margem para picos, mas não substitui RAM física. Os ajustes abaixo são um ponto inicial para um laboratório de baixo uso, não um dimensionamento para produção com muitos usuários.


O Nginx será instalado no host Ubuntu somente na etapa de HTTPS. Assim, `127.0.0.1:8080` continuará apontando para o GLPI, e o banco continuará sem porta publicada na VM.

## 1. Diagnóstico inicial

Executar no Ubuntu por SSH, fora dos containers:

```bash
free -h
sudo swapon --show
df -h /
findmnt -no FSTYPE /
docker --version
docker compose version
sudo docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
sudo ss -lntup
```

Confirmar espaço livre suficiente e filesystem ext4 ou XFS. Para Btrfs, não utilizar diretamente este procedimento de swap.

No laboratório, não havia swap, existiam aproximadamente 188 GB livres e as portas 80, 443 e 8080 estavam disponíveis. A ausência de portas na coluna `PORTS` do RustDesk não significa que ele não escuta na VM: uma configuração com rede do host também pode produzir esse resultado. Conferir com `ss` e, se necessário, `docker inspect`.


## 2. Criar swap de 4 GiB
### 2.1 Esse comando cria um arquivo vazio de 4 GB
```bash
sudo dd if=/dev/zero of=/swapfile-glpi bs=1M count=4096 status=progress conv=excl
```
Executar um comando de cada vez; não continuar se algum falhar:

```bash
sudo chmod 600 /swapfile-glpi
sudo mkswap /swapfile-glpi
sudo swapon /swapfile-glpi
sudo swapon --show
free -h
```

Resultado esperado: `/swapfile-glpi` com tamanho `4G`. O uso inicial em zero é normal.

### 2.2 Tornar persistente

```bash
sudo cp -a /etc/fstab "/etc/fstab.backup-$(date +%Y%m%d-%H%M%S)"
```

```bash
grep -qE '^[[:space:]]*/swapfile-glpi[[:space:]]' /etc/fstab || echo '/swapfile-glpi none swap sw 0 0' | sudo tee -a /etc/fstab
```

```bash
sudo findmnt --verify --verbose
```

Resolver eventuais erros antes de prosseguir. Então:

```bash
sudo systemctl daemon-reload
sudo swapon -a
grep '/swapfile-glpi' /etc/fstab
```

Entrada esperada:

```text
/swapfile-glpi none swap sw 0 0
```


## 3. Preparar o projeto GLPI

```bash
mkdir -p ~/glpi
cd ~/glpi
```

### 3.1 Gerar credenciais

```bash
(
  umask 077
  set -o noclobber
  {
    printf 'GLPI_DB_PASSWORD=%s\n' "$(openssl rand -hex 24)"
    printf 'MARIADB_ROOT_PASSWORD=%s\n' "$(openssl rand -hex 24)"
  } > .env
)
```

### 3.2 Ajustar PHP

```bash
cat > php-custom.ini <<'EOF'
memory_limit = 256M
date.timezone = America/Sao_Paulo
upload_max_filesize = 20M
post_max_size = 24M
EOF
```

### 3.3 Ajustar processos do Apache

```bash
cat > apache-small.conf <<'EOF'
<IfModule mpm_prefork_module>
    StartServers 1
    MinSpareServers 1
    MaxSpareServers 2
    MaxRequestWorkers 3
    MaxConnectionsPerChild 500
</IfModule>
EOF
```

## 4. Criar o Docker Compose

Salvar como `~/glpi/compose.yaml`:

```yaml
name: glpi

services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MARIADB_DATABASE: glpi
      MARIADB_USER: glpi
      MARIADB_PASSWORD: ${GLPI_DB_PASSWORD}
      TZ: America/Sao_Paulo
    command:
      - --innodb-buffer-pool-size=128M
      - --max-connections=20
      - --performance-schema=OFF
      - --tmp-table-size=16M
      - --max-heap-table-size=16M
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 30
      start_period: 60s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  glpi:
    image: glpi/glpi:11.0.12
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      GLPI_DB_HOST: db
      GLPI_DB_PORT: "3306"
      GLPI_DB_NAME: glpi
      GLPI_DB_USER: glpi
      GLPI_DB_PASSWORD: ${GLPI_DB_PASSWORD}
      GLPI_SKIP_AUTOUPDATE: "true"
      TZ: America/Sao_Paulo
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - glpi_data:/var/glpi
      - ./php-custom.ini:/usr/local/etc/php/conf.d/zz-custom.ini:ro
      - ./apache-small.conf:/etc/apache2/conf-enabled/zz-small.conf:ro
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  db_data:
  glpi_data:
```

Pode-se criar o arquivo com `nano compose.yaml`, colar o YAML, salvar com `Ctrl+O`, confirmar com Enter e sair com `Ctrl+X`.


## 5. Iniciar e validar

```bash
cd ~/glpi
sudo docker compose config --quiet
```

Se não houver erro:

```bash
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=80 glpi
sudo docker stats --no-stream
```

A primeira inicialização pode levar alguns minutos. O banco deve apresentar `healthy`. Aguardar no log do GLPI:

```text
GLPI installation completed successfully!
```

Testar:

```bash
curl -I --max-time 15 http://127.0.0.1:8080
```

No ambiente validado, a resposta foi `HTTP/1.1 200 OK`; Apache e `glpi-cron` entraram em estado `RUNNING`. A mensagem isolada `Test logger` durante a instalação não impediu a conclusão.


## 6. Acessar pelo PuTTY

1. Conectar à VM com o PuTTY e a chave `.ppk` habitual.
2. Na barra de título, clicar com o botão direito e escolher **Change Settings…**.
3. Abrir **Connection → SSH → Tunnels**.
4. Preencher os campos abaixo.

| Campo | Valor |
|---|---|
| Source port | `8080` |
| Destination | `127.0.0.1:8080` |
| Tipo | Local |
| Família | Auto |
| Local ports accept connections from other hosts | Desmarcado |

5. Clicar em **Add** e conferir `L8080 127.0.0.1:8080`.
6. Clicar em **Apply** e manter a sessão conectada.
7. No Navegador, abrir [http://localhost:8080](http://localhost:8080).

Se 8080 já estiver ocupada no Windows, utilizar Source port `8081`, manter Destination `127.0.0.1:8080` e acessar `http://localhost:8081`.

<img width="1366" height="728" alt="image" src="https://github.com/user-attachments/assets/bcf62c7d-4d36-485e-abd3-5c42d95d307a" />


## 7. Configurar domínio e HTTPS


Este procedimento usa Nginx no Ubuntu como proxy reverso e Certbot para certificado Let's Encrypt. O TLS termina no Nginx; o tráfego para o container usa a porta local 8080. Não instalar outro proxy que dispute as portas 80 e 443.


### 7.1 Liberar a rede na Oracle

Na Console OCI, localizar a instância, sua VNIC e a subnet. Identificar os **Network Security Groups (NSGs)** associados e/ou as **Security Lists** da subnet que controlam o tráfego.

Adicionar regras de ingresso **stateful**, com porta de origem qualquer:

| Origem | Protocolo | Porta de destino | Uso |
|---|---|---|---|
| `0.0.0.0/0` | TCP | 80 | Validação HTTP do certificado e redirecionamento |
| `0.0.0.0/0` | TCP | 443 | Acesso HTTPS |

Manter as regras existentes de SSH e RustDesk. Não liberar 3306 ou 8080. A VM precisa de IP público, rota adequada para Internet Gateway e saída para DNS e HTTPS. Não é necessário liberar UDP 443 para este procedimento com Nginx.

### 7.2 Conferir o firewall do Ubuntu

```bash
sudo ufw status verbose
sudo iptables -S INPUT
sudo nft list ruleset
```

Usar o gerenciador que já controla o firewall da VM. Não habilitar UFW por impulso, não limpar regras e não substituir o ruleset da Oracle ou do Docker.

Se **UFW já estiver ativo**:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Se UFW estiver inativo e houver bloqueio explícito na cadeia INPUT do iptables, após revisar as regras, pode-se adicionar somente as duas permissões antes do bloqueio:

```bash
sudo iptables -C INPUT -p tcp --dport 80 -j ACCEPT || sudo iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT
sudo iptables -C INPUT -p tcp --dport 443 -j ACCEPT || sudo iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
```

Esses dois comandos são temporários. Se a VM já usa `netfilter-persistent`, persistir conforme esse mecanismo:

```bash
sudo netfilter-persistent save
```


### 7.3 Instalar Nginx e Certbot

Confirmar que 80 e 443 continuam livres:

```bash
sudo ss -lntp
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

Este guia utiliza os pacotes APT do Ubuntu; não instalar simultaneamente uma segunda versão do Certbot via snap. Conferir a orientação atual do Certbot antes de reproduzir em outra versão do sistema.

Criar o site:

```bash
sudo nano /etc/nginx/sites-available/glpi
```

Conteúdo, substituindo o domínio:

```nginx
server {
    listen 80;
    server_name glpi.exemplo.com.br;

    client_max_body_size 24m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

Este exemplo pressupõe conexão direta da internet ao Nginx, sem CDN à frente. O cabeçalho de protocolo é sobrescrito pelo proxy. O container continua publicado somente em `127.0.0.1`.

Ativar o site novo, testar e recarregar:

```bash
sudo ln -s /etc/nginx/sites-available/glpi /etc/nginx/sites-enabled/glpi
sudo nginx -t
```

Se o link já existir, verificar seu destino em vez de sobrescrevê-lo. Se o teste passar:

```bash
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

Testar o virtual host na VM:

```bash
curl -I -H 'Host: glpi.exemplo.com.br' http://127.0.0.1
```


### 7.4 Reconhecer HTTPS no Apache do container

Como o Nginx encaminha a requisição por HTTP local, o Apache precisa reconhecer o protocolo original. Adicionar **uma única vez** ao final de `~/glpi/apache-small.conf`:

```apache
SetEnvIf X-Forwarded-Proto "^https$" HTTPS=on
```

Validar a configuração e reiniciar somente o serviço GLPI:

```bash
cd ~/glpi
sudo docker compose exec glpi apache2ctl -t
sudo docker compose restart glpi
```

Prosseguir somente se a validação indicar `Syntax OK`. Não expor a porta do container em `0.0.0.0`, pois esse modelo confia no cabeçalho definido pelo proxy local.

### 7.5 Emitir o certificado e redirecionar HTTP

Com DNS correto, Nginx funcionando e TCP 80 acessível externamente:

```bash
sudo certbot --nginx -d glpi.exemplo.com.br --redirect
```

Informar um e-mail válido e aceitar os termos quando solicitado. O Certbot ajusta o virtual host para HTTPS e instala o certificado. Se a emissão falhar, corrigir DNS/rede antes de repetir muitas vezes, evitando limites de tentativas da autoridade certificadora.

Conferir:

```bash
sudo nginx -t
sudo certbot certificates
curl -I https://glpi.exemplo.com.br
curl -I http://glpi.exemplo.com.br
```

Esperado: HTTPS com certificado válido, sem necessidade de `-k`, e HTTP redirecionando para HTTPS. Abrir pelo navegador de outro computador sem túnel e confirmar login, navegação e ausência de redirecionamentos para `localhost`.

Na configuração geral do GLPI, ajustar o campo de URL da aplicação, quando exibido, para `https://glpi.exemplo.com.br`. Confirmar o endereço também em links de notificações e integrações.

Depois de migrar o acesso para HTTPS, acrescentar ao `php-custom.ini`:

```ini
session.cookie_secure = On
session.cookie_httponly = On
session.cookie_samesite = Lax
```

Reiniciar apenas o GLPI e testar novamente. Com cookies marcados como Secure, utilizar HTTPS como endereço administrativo; o acesso HTTP pelo túnel pode deixar de manter a sessão. Para voltar temporariamente ao túnel, reverter `session.cookie_secure` e reiniciar o GLPI, mantendo o serviço público protegido.

### 7.6 Renovação automática

```bash
sudo systemctl status certbot.timer --no-pager
sudo systemctl list-timers --all
sudo certbot renew --dry-run
```

Na instalação APT, conferir que existe agendamento de renovação. Se o timer estiver instalado e desabilitado, ativar:

```bash
sudo systemctl enable --now certbot.timer
```

Criar um hook de recarga após renovação bem-sucedida:

```bash
sudo install -d -m 755 /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh > /dev/null <<'EOF'
#!/bin/sh
/usr/sbin/nginx -t && /usr/bin/systemctl reload nginx
EOF
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Testar o hook manualmente e repetir o teste de renovação. Manter DNS e porta 80 disponíveis para o método HTTP-01 usado aqui. O teste `--dry-run` não garante que todo futuro problema de DNS ou firewall será evitado.

### 7.7 Inventário

Após HTTPS validado, habilitar o inventário no GLPI e instalar o GLPI Agent nos computadores. Para o inventário nativo, o endereço do servidor pode ser definido como:

```text
https://glpi.exemplo.com.br/front/inventory.php
```

## 8. Operação, backup e atualização

### Monitoramento

```bash
cd ~/glpi
free -h
sudo swapon --show
sudo docker stats --no-stream
sudo docker compose ps
sudo docker compose logs --tail=100 glpi
sudo docker compose logs --tail=100 db
vmstat 2 10
df -h /
```


### Backup

Guardar, fora da VM, uma cópia protegida de:

- Dump consistente do MariaDB.
- Conteúdo completo do volume montado em `/var/glpi`, incluindo configuração e chaves de criptografia.
- `compose.yaml`, `.env`, `php-custom.ini` e `apache-small.conf`.
- Após HTTPS, configuração do Nginx e `/etc/letsencrypt`, incluindo chaves privadas.

Exemplo de dump com pausa breve do GLPI para reduzir alterações durante a cópia:

```bash
cd ~/glpi
mkdir -p backups
chmod 700 backups
sudo docker compose stop glpi
```

```bash
(
  umask 077
  set -o noclobber
  sudo docker compose exec -T db sh -c 'exec mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --single-transaction --routines --events --triggers glpi' > "backups/glpi-$(date +%Y%m%d-%H%M%S).sql"
)
```

Confirmar saída sem erro e arquivo não vazio. Para copiar os dados do GLPI enquanto ele está parado:

```bash
backup_dir="$(pwd)/backups/arquivos-$(date +%Y%m%d-%H%M%S)"
mkdir -m 700 "$backup_dir"
sudo docker cp "$(sudo docker compose ps -a -q glpi):/var/glpi/." "$backup_dir/"
```

Reiniciar o GLPI **mesmo se uma cópia falhar**, depois investigar e refazer o backup:

```bash
sudo docker compose start glpi
sudo docker compose ps
```


### Atualizações

1. Ler notas de versão, requisitos e compatibilidade de plugins.
2. Fazer backup e testar recuperação antes de uma mudança relevante.
3. Alterar a tag do GLPI explicitamente no Compose, evitando saltos de versão não planejados.
4. Baixar a imagem e recriar somente o serviço necessário em janela de manutenção.
5. Como `GLPI_SKIP_AUTOUPDATE=true`, executar a atualização do banco conforme a documentação da versão. Não considerar a troca de imagem suficiente.
6. Validar logs, login, tarefas automáticas e inventário.



## Referências oficiais

- [Imagem Docker oficial do GLPI](https://github.com/glpi-project/docker-images)
- [Tags publicadas do GLPI](https://hub.docker.com/r/glpi/glpi/tags)
- [Pré-requisitos do GLPI](https://glpi-install.readthedocs.io/en/latest/prerequisites.html)
- [Documentação do GLPI Agent](https://glpi-agent.readthedocs.io/en/latest/)
- [Healthcheck do MariaDB em Docker](https://mariadb.com/docs/server/server-management/automated-mariadb-deployment-and-administration/docker-and-mariadb/using-healthcheck-sh)
- [Docker Compose](https://docs.docker.com/compose/)
- [Swap no Ubuntu — swapon](https://manpages.ubuntu.com/manpages/jammy/man8/swapon.8.html)
- [Oracle Cloud — regras de segurança](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm)
- [Nginx — proxy reverso](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Apache — SetEnvIf](https://httpd.apache.org/docs/2.4/mod/mod_setenvif.html)
- [Certbot — instruções de instalação](https://certbot.eff.org/instructions)
- [Certbot — uso e renovação](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Let's Encrypt — métodos de validação](https://letsencrypt.org/docs/challenge-types/)
- [Manual do PuTTY — túneis](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-ssh-portfwd)
