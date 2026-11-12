# Notas SEI lab

Dificuldades: as permissões do manual (`chown root.apache` e os `chmod` das pastas e do `temp`). precisam ser rodadas com o comando sudo.

Houve dificuldade para configurar o php por causa da falta das bibliotecas que o PHP exige e que não estavam habilitadas: `libargon2`, `libraqm`, `libimagequant` e `libavif`. Elas ficam nos repositórios **EPEL** e **CodeReady Builder** (no Oracle Linux, `ol8_codeready_builder`).

Houve dificuldade para configurar o local host, era necessário utilizar o localhot:127.0.0.1

Houve também dificuldade para configurar o SIP e o SEI para ficarem acessíveis

Tive problemas de permissão, era necessário usar o comando sudo su, para ter acesso de super usuário no bash.

Foi identificado um erro que dependendo do arquivo que é anexado ao protocolo não é possível transformar aquele protocolo em PDF.

Tive problemas com a funcionalidade relacionado a pesquisa

![image.png](image.png)

ficava retornando este erro, parecia estar relacionado a um erro de numero 12

### 1. Confirme se o Solr existe

bash

```bash
sudo systemctl status solr --no-pager | head -5sudo ss -tlnp | grep 8983curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8983/solr/
```

Se disser `Unit solr.service could not be found`, não aparecer nada na porta 8983 e o `curl` der `000`, é isso. Se o Solr estiver rodando, pule para o item 4.

### 2. Instale o Solr (seção 21 do manual)

A VM tem cerca de 7,5 GB de RAM, e MySQL, Apache e PHP dividem essa memória. Por isso uso 2 GB de heap em vez dos 6 GB do manual.

bash

```bash
sudo dnf install -y java-1.8.0-openjdkjava -version 2>&1 | head -1sudo useradd solrcd /tmpcurl -fLO https://archive.apache.org/dist/lucene/solr/8.2.0/solr-8.2.0.tgzls -lh solr-8.2.0.tgzsudo cp -r /opt/sei/config/solr/. /tmp/ls /tmp
```

O `.tgz` deve ter cerca de 190 MB. Se o `curl` falhar, a VM talvez não alcance o site da Apache,. O `ls /tmp` deve mostrar `log4j.properties`, `sei-solr-8.2.0.sh`, `sei-cores-8.2.0` e `solr.service`.

Antes de rodar o script, leia o conteúdo, porque ele roda como root:

bash

```bash
sudo cat /tmp/sei-solr-8.2.0.sh
```

Se instalar o Solr em `/opt/solr` e criar os índices em `/dados`, como o manual descreve, execute:

bash

```bash
sudo bash /tmp/sei-solr-8.2.0.shsudo ls /dados
```

Devem existir `sei-protocolos`, `sei-bases-conhecimento` e `sei-publicacoes`.

### 3. Ajuste a memória, restrinja ao local e suba o serviço

bash

```bash
sudo sed -i 's|^#\?SOLR_JAVA_MEM=.*|SOLR_JAVA_MEM="-Xms512m -Xmx2048m"|' /opt/solr/bin/solr.in.shecho 'SOLR_JETTY_HOST="127.0.0.1"' | sudo tee -a /opt/solr/bin/solr.in.shsudo systemctl daemon-reloadsudo systemctl enable --now solrsleep 20sudo systemctl status solr --no-pager | head -5curl -s 'http://127.0.0.1:8983/solr/admin/cores?action=STATUS' | head -c 400
```

O `SOLR_JETTY_HOST="127.0.0.1"` faz o Solr escutar só nesta máquina. Isso substitui a lista de IPs do `jetty.xml` (passo 9 do manual), que é justamente o ponto do problema 12, e é mais simples e seguro, já que o SEI está no mesmo servidor. O `curl` deve listar os três índices.

Confirme que o `ConfiguracaoSEI.php` tem o bloco `Solr` com `http://127.0.0.1:8983/solr` e os três `Core...` (já está assim).

### 4. Indexe o que já existe no banco

O Solr novo começa vazio. Sem essa etapa, a pesquisa não retorna nada:

bash

```bash
sudo php -c /etc/php.ini /opt/sei/scripts/indexacao_protocolos_completa.phpsudo php -c /etc/php.ini /opt/sei/scripts/indexacao_publicacoes.phpsudo php -c /etc/php.ini /opt/sei/scripts/indexacao_bases_conhecimento.php
```

Com o banco de referência, deve ser rápido. Os documentos criados daqui para frente são indexados pelo agendamento (cron do SEI), então confirme que o `/etc/cron.d/sei` existe e está rodando (`sudo tail /var/log/infra_agendamento_sei.log`). Os índices só refletem as alterações em até 5 minutos (`TempoCommitProtocolos`).

### 5. Teste

Reinicie e pesquise de novo no SEI:

bash

```bash
sudo systemctl restart php-fpm httpd
```

Procure por um termo que exista em um processo ou documento, como o número do protocolo ou uma palavra do texto. Se a mensagem persistir, rode `sudo tail -15 /var/log/php-fpm/www-error.log` logo depois e olhe **Infra → Log** no SEI. Esses logs mostram o erro de conexão com o Solr, se houver.

### Se o Solr já estava rodando

Nesse caso a causa pode ser outra, e o problema 12 se aplica:

- confira o `Servidor` e os nomes dos `Core` na seção `Solr` do `ConfiguracaoSEI.php`
- veja o log do Solr: `sudo tail -30 /var/solr/logs/solr.log` (o caminho pode variar, `sudo find / -name solr.log 2>/dev/null` acha)
- confirme que `/tmp` tem 2 GB livres (`df -h /tmp`)
- se o Solr tem `jetty.xml` com limitação de IPs, `127.0.0.1` precisa estar na lista

Se falhar 

O script não rodou nada: `/dados` e `/opt/solr` não existem. O motivo são os `$'\r'` do erro. Os arquivos de configuração do pacote vêm com quebra de linha de Windows (CRLF), e o bash e o systemd não aceitam isso. A VM, o download e o Java estão certos (o `.tgz` baixou com 174 MB).

A indexação que apareceu como “10 protocolos indexados em 0.051s” não vale, porque o Solr não existia. Depois que ele subir, é preciso rodar de novo.

### 1. Corrija as quebras de linha

Os três arquivos que o script usa:

bash

```bash
sudo sed -i 's/\r$//' /tmp/sei-solr-8.2.0.sh /tmp/solr.service /tmp/log4j.propertiesgrep -c $'\r' /tmp/sei-solr-8.2.0.sh /tmp/solr.service /tmp/log4j.properties
```

O `grep -c` deve dar `0` nos três. Confira também o serviço que será instalado:

bash

```bash
cat /tmp/solr.service
```

Ele deve usar `User=solr` e caminhos em `/opt/solr`. 

### 2. Rode o script

bash

```bash
sudo bash /tmp/sei-solr-8.2.0.shls /dados /opt/solr/bin/solr.in.shls /etc/systemd/system/solr.service
```

Em `/dados` devem aparecer `sei-protocolos`, `sei-bases-conhecimento` e `sei-publicacoes`. Se o script reclamar de `mkdir: cannot create directory '/dados': File exists`, é porque rodou duas vezes.

### 3. Memória e acesso só local

bash

```bash
sudo sed -i 's|^#\?SOLR_JAVA_MEM=.*|SOLR_JAVA_MEM="-Xms512m -Xmx2048m"|' /opt/solr/bin/solr.in.shecho 'SOLR_JETTY_HOST="127.0.0.1"' | sudo tee -a /opt/solr/bin/solr.in.shgrep -n -E '^SOLR_JAVA_MEM|^SOLR_JETTY_HOST' /opt/solr/bin/solr.in.sh
```

O `grep` deve mostrar as duas linhas sem `#` na frente. Se `SOLR_JAVA_MEM` não aparecer, acrescente com `echo 'SOLR_JAVA_MEM="-Xms512m -Xmx2048m"' | sudo tee -a /opt/solr/bin/solr.in.sh`.

### 4. Suba o serviço

bash

```bash
sudo systemctl daemon-reloadsudo systemctl enable --now solrsleep 30sudo systemctl status solr --no-pager | head -6curl -s 'http://127.0.0.1:8983/solr/admin/cores?action=STATUS' | head -c 400
```

O serviço deve estar `active (running)`, e o `curl` deve listar os três índices. Se o serviço falhar:

bash

```bash
sudo journalctl -u solr -n 30 --no-pagersudo ausearch -m avc -ts recent | tail -10
```

O segundo mostra se o SELinux bloqueou (o servidor está em `Enforcing`). Me mande a saída dos dois.

### 5. Indexe de novo, agora com o Solr no ar

bash

```bash
sudo php -c /etc/php.ini /opt/sei/scripts/indexacao_protocolos_completa.phpsudo php -c /etc/php.ini /opt/sei/scripts/indexacao_publicacoes.phpsudo php -c /etc/php.ini /opt/sei/scripts/indexacao_bases_conhecimento.phpsleep 10curl -s 'http://127.0.0.1:8983/solr/sei-protocolos/select?q=*:*&rows=0' | grep -o '"numFound":[0-9]*'
```

O `numFound` deve ser maior que `0`. Se for `0`, o `Solr` está de pé mas não recebeu os dados, e o `Infra → Log` do SEI mostra o motivo.

### 6. Teste a pesquisa

bash

```bash
sudo systemctl restart php-fpm httpd
```

Pesquise no SEI por um número de protocolo ou uma palavra de um documento. Os índices refletem alterações em até 5 minutos (`TempoCommitProtocolos`).

Se a mensagem “Erro realizando pesquisa” continuar com o Solr funcionando, rode `sudo tail -15 /var/log/php-fpm/www-error.log` logo depois da tentativa e olhe o **Infra → Log**.

## 0. Confirme o ambiente

```bash
cat /etc/os-release
sudo whoami
```

Se o `sudo` funcionar, você consegue seguir. A versão do SO muda alguns comandos (por exemplo, `yum` e `dnf`).

## 1. Descompacte o pacote

```bash
cd /opt/sei-lab
sudo dnf install -y unzip
sudo unzip sei-4.0.12.15.zip -d /opt/sei-lab/pacote
ls -R /opt/sei-lab/pacote | head -50
```

Veja o que veio dentro: fontes (`sei`, `sip`, `infra`) e os dumps do banco (`SEI-BD-...`). Se vierem separados, você precisa dos dois.

## 2. Instale as dependências (seção 2 do manual)

- **Apache** (httpd)
- **PHP 7.3** com as extensões listadas: mysqli (ou o driver do seu banco), gd, bcmath, soap, mbstring, ldap, intl, zip, memcache, uploadprogress etc. No Oracle Linux/RHEL 8 o PHP 7.3 normalmente vem do repositório Remi.
- **Banco de dados** (MySQL, ou o que o lab usar). Se for MySQL 8, veja o item 32 do manual (`mysql_native_password`).
- **Memcached**
- **Java 8**, **wkhtmltopdf 0.12.6**, **ffmpeg** e fontes TrueType

```bash
sudo dnf install -y httpd memcached java-1.8.0-openjdk
sudo systemctl enable --now httpd memcached
```

## 3. Coloque os fontes em /opt (seção 3)

```bash
sudo mkdir -p /opt/sei /opt/sip /opt/infra
sudo cp -r /opt/sei-lab/pacote/sei/*   /opt/sei/
sudo cp -r /opt/sei-lab/pacote/sip/*   /opt/sip/
sudo cp -r /opt/sei-lab/pacote/infra/* /opt/infra/
```

Depois aplique as permissões do manual (`chown root.apache` e os `chmod` das pastas e do `temp`).

```bash
#SEI
chown -R root.apache /opt/sei
find /opt/sei -type d -exec chmod 2750 {} \; find /opt/sei -type f -exec chmod 0640 {} \; find /opt/sei/temp -type d -exec chmod 2570 {} \;

#SIP
chown -R root.apache /opt/sip
find /opt/sip -type d -exec chmod 2750 {} \; find /opt/sip -type f -exec chmod 0640 {} \; find /opt/sip/temp -type d -exec chmod 2570 {} \;

#Infra PHP
chown -R root.apache /opt/infra
find /opt/infra -type d -exec chmod 2750 {} \; find /opt/infra -type f -exec chmod 0640 {} \;
```

## 4. Configure o Apache e o PHP

- Crie `/etc/httpd/conf.d/sei.conf` com o conteúdo da seção 3 do manual (os Alias e as Directory).

```bash
sudo tee /etc/httpd/conf.d/sei.conf > /dev/null <<'EOF'
Alias "/sei" "/opt/sei/web"
Alias "/sip" "/opt/sip/web"
Alias "/infra_css" "/opt/infra/infra_css"
Alias "/infra_js" "/opt/infra/infra_js"

<Directory />
AllowOverride None
Require all denied
</Directory>

<Directory ~ "(/opt/sei/web|/opt/sip/web|/opt/infra/infra_css|/opt/infra/infra_js)" >
AllowOverride None
Options None
Require all granted
</Directory>
EOF
```

Confira:

```bash
cat /etc/httpd/conf.d/sei.conf
sudo apachectl configtest
```

O esperado é `Syntax OK`.

- Ajuste o `php.ini` com a tabela da seção 2: `short_open_tag=1`, `default_charset=ISO-8859-1`, `include_path=/opt/infra/infra_php` etc.

### 1. Instale e habilite o EPEL

```bash
sudo dnf install -y oracle-epel-release-el8
sudo dnf config-manager --set-enabled ol8_developer_EPEL
sudo dnf makecache
dnf repolist | grep -i epel
```

Você deve ver `ol8_developer_EPEL` na lista. Se o primeiro comando disser que o pacote não existe, use o EPEL oficial

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf makecache
dnf repolist | grep -i epel
```

### 2. Confirme que as bibliotecas aparecem

```bash
dnf provides '/libargon2.so.1' '/libraqm.so.0' '/libimagequant.so.0' '/libavif.so.14' | grep -i epel
```

### 3. Instale as bibliotecas e o PHP

```bash
sudo dnf install -y libargon2 libraqm libimagequant libavif

sudo dnf install -y php php-cli php-common php-mysqlnd php-gd php-bcmath \
php-soap php-mbstring php-ldap php-intl php-xml php-zip php-pecl-zip \
php-pecl-memcache php-process php-opcache
```

### 4. Confirme

```bash
php -v
php --ini
```

O esperado é `PHP 7.3.33` e `Loaded Configuration File: /etc/php.ini`.

- Se o SELinux estiver ativo, rode os `setsebool -P ...` do manual.

Veja se está ativo:

```bash
getenforce
```

- `Disabled` ou `Permissive`: pule esta parte (mas `Permissive` ainda registra avisos).
- `Enforcing`: rode os comandos do manual. Atenção que o PDF original tem hífen “longo” (`–P`), que não funciona no terminal. Use o hífen normal:

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_memcache 1
sudo setsebool -P httpd_execmem 1
sudo setsebool -P httpd_can_connect_ldap 1
sudo setsebool -P httpd_use_nfs 1
sudo setsebool -P httpd_can_network_connect_db 1
```

Se algum comando reclamar que o booleano não existe, pode ignorar esse e seguir.

### 4.4 Reiniciar e testar

```bash
sudo systemctl restart httpd
sudo systemctl status httpd --no-pager
```

Se o PHP for do Remi (php-fpm), reinicie também:

```bash
sudo systemctl restart php73-php-fpm
```

Teste rápido de que o PHP está lendo as configurações:

```bash
php -i | grep -E 'short_open_tag|default_charset|include_path’
```

Se algo falhar, veja o log:

```bash
sudo tail -30 /var/log/httpd/error_log
```

## 5. Banco de dados

### 5.1. Verifique se o MySQL está instalado

```bash
mysql --version
sudo systemctl status mysqld --no-pager
```

Se não estiver:

### 5.1.1. Veja os streams disponíveis

```bash
dnf module list mysql
```

Você pode encontrar, por exemplo, `8.0` (AppStream) e `8.4-community` (repositório MySQL). Use o stream disponível no ambiente e siga **uma** das opções abaixo.

### Opção A — MySQL Community 8.4

```bash
sudo dnf module reset mysql -y
sudo dnf module enable mysql:8.4-community -y
sudo dnf install -y mysql-community-server
mysql --version
```

### Opção B — MySQL 8.0 do AppStream

```bash
sudo dnf module reset mysql -y
sudo dnf module enable mysql:8.0 -y
sudo dnf install -y mysql-server
mysql --version
```

### 5.2. Configure o charset e inicie o serviço

Crie `/etc/my.cnf.d/sei.cnf`:

```bash
sudo tee /etc/my.cnf.d/sei.cnf > /dev/null <<'EOF'
[mysqld]
character-set-server=latin1
collation-server=latin1_swedish_ci

[client]
default-character-set=latin1
EOF
```

Inicie o serviço e confira o status:

```bash
sudo systemctl enable --now mysqld
sudo systemctl status mysqld --no-pager
```

O esperado é `active (running)`. Se o serviço falhar, consulte o log:

```bash
sudo tail -30 /var/log/mysqld.log
```

### 5.3. Defina a senha do root

No prompt do MySQL, defina uma senha forte e depois saia:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SuaSenhaForte#2026';
EXIT;
```

A senha deve conter letras maiúsculas e minúsculas, número e símbolo. Anote-a em um local seguro.

Confira a versão e as configurações de charset:

```bash
mysql -u root -p -e "SELECT VERSION(); SHOW VARIABLES LIKE 'character_set%';"
```

`character_set_server` e `character_set_client` devem aparecer como `latin1`. Se aparecer `utf8mb4`, o `sei.cnf` não foi lido: confira com `cat /etc/my.cnf.d/sei.cnf` e reinicie com `sudo systemctl restart mysqld`.

### 5.4. Localize os dumps

```bash
find /opt/sei-lab -iname '*BD*' -o -iname '*.sql' | head -20
```

### 5.5. Descompacte o dump do MySQL

```bash
sudo mkdir -p /opt/sei-lab/bd
sudo unzip /opt/sei-lab/pacotes/sei-bd-mysql-4.0.12.15.zip -d /opt/sei-lab/bd
sudo find /opt/sei-lab/bd -type f -exec ls -lh {} +
```

### 5.6. Inspecione os dumps

Troque `ARQUIVO` pelos nomes reais que apareceram:

```bash
sudo head -40 /opt/sei-lab/bd/ARQUIVO_SIP.sql
sudo grep -n -i -E 'CREATE DATABASE|^USE ' /opt/sei-lab/bd/*.sql | head
```

### 5.7. Crie os bancos e importe os dados

Se os dumps **não** tiverem instruções `CREATE DATABASE`, siga estas etapas:

```bash
mysql -u root -p -e "CREATE DATABASE sip CHARACTER SET latin1; CREATE DATABASE sei CHARACTER SET latin1;"
sudo cat /opt/sei-lab/bd/ARQUIVO_SIP.sql | mysql -u root -p sip
sudo cat /opt/sei-lab/bd/ARQUIVO_SEI.sql | mysql -u root -p sei
```

Se os dumps **já** tiverem as instruções `CREATE DATABASE`, importe sem indicar o banco na linha de comando:

```bash
sudo cat /opt/sei-lab/bd/ARQUIVO.sql | mysql -u root -p
```

### 5.8. Confira os bancos importados

```bash
mysql -u root -p -e "SHOW DATABASES; SELECT COUNT(*) FROM sip.orgao; SELECT COUNT(*) FROM sei.orgao;"
```

Os dois resultados de `COUNT(*)` devem ser pelo menos `1`.

### Exemplo com nomes de arquivos específicos

```bash
ls -lh /opt/sei-lab/bd
mysql -u root -p -e "CREATE DATABASE sip CHARACTER SET latin1; CREATE DATABASE sei CHARACTER SET latin1;"
sudo cat /opt/sei-lab/bd/sip_4.0.x.1_BD_Ref_Exec.sql | mysql -u root -p sip
sudo cat /opt/sei-lab/bd/sei_4.0.x.1_BD_Ref_Exec.sql | mysql -u root -p sei
mysql -u root -p -e "SHOW DATABASES; SELECT COUNT(*) FROM sip.orgao; SELECT COUNT(*) FROM sei.orgao; SELECT COUNT(*) FROM sip.usuario;"
```

### 5.9. Erros comuns na importação

Os erros mais prováveis ao trazer um dump de MariaDB para MySQL 8 são:

- `Incorrect datetime value` ou `Invalid default value`: problema de `sql_mode`. Me mande a mensagem.
- `Unknown collation` ou `Unknown character set`: me mande a mensagem.
- `Access denied`: a senha do root ainda não foi definida. Entre com `sudo mysql -u root` e rode `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SuaSenha';`.

Se o import falhar no meio, apague e recrie o banco antes de repetir:

```bash
mysql -u root -p -e "DROP DATABASE sip; CREATE DATABASE sip CHARACTER SET latin1;"
```

### 5.10. Defina as variáveis

Escolha as senhas e informe o endereço usado para acessar o sistema. Evite `$` e aspas nas senhas:

```bash
SENHA_SIP='SenhaSip#2026'
SENHA_SEI='SenhaSei#2026'
SERVIDOR='152.70.210.189'
SIGLA='LAB'
DESCRICAO='Laboratorio SEI'
```

Em `SERVIDOR`, use o IP ou hostname digitado no navegador. Em uma VM na nuvem, o servidor pode não alcançar o próprio IP público. Se aparecer “Erro acessando arquivo WSDL”, tente o IP privado retornado por `hostname -I`.

### 5.11. Crie os usuários da aplicação sem permissões DDL

```bash
mysql -u root -p <<EOF
CREATE USER 'sip_user'@'localhost' IDENTIFIED WITH mysql_native_password BY '${SENHA_SIP}';
CREATE USER 'sei_user'@'localhost' IDENTIFIED WITH mysql_native_password BY '${SENHA_SEI}';
GRANT SELECT, INSERT, UPDATE, DELETE ON sip.* TO 'sip_user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON sei.* TO 'sei_user'@'localhost';
FLUSH PRIVILEGES;
EOF
```

### 5.12. Atualize os dados do órgão e do sistema

```bash
mysql -u root -p <<EOF
USE sip;
UPDATE orgao SET sigla='${SIGLA}', descricao='${DESCRICAO}' WHERE id_orgao=0;
UPDATE sistema SET pagina_inicial='http://${SERVIDOR}/sip' WHERE sigla='SIP';
UPDATE sistema
SET pagina_inicial='http://${SERVIDOR}/sei/inicializar.php',
    web_service='http://${SERVIDOR}/sei/controlador_ws.php?servico=sip'
		WHERE sigla='SEI';

USE sei;
UPDATE orgao SET sigla='${SIGLA}', descricao='${DESCRICAO}' WHERE id_orgao=0;
EOF
```

Se o shell ficar aguardando mais linhas, confira se `EOF` está sozinho na linha, sem espaços.

Confira os dados gravados:

```bash
mysql -u root -p -e "SELECT id_orgao, sigla, descricao FROM sip.orgao; SELECT id_orgao, sigla, descricao FROM sei.orgao; SELECT sigla, pagina_inicial, web_service FROM sip.sistema;"
```

A sigla deve ser igual nos dois `orgao`, e as páginas do SEI e do SIP devem estar com o seu endereço.

Teste também se os usuários da aplicação conseguem entrar:

```bash
mysql -u sip_user -p -e "SELECT COUNT(*) FROM sip.orgao;"
mysql -u sei_user -p -e "SELECT COUNT(*) FROM sei.orgao;"
```

### 5.13. Confira os diretórios da aplicação

```bash
ls /opt/sei /opt/sip /opt/infra
```

Se aparecer `Permission denied` em `/opt/sei`, isso pode ocorrer porque a pasta pertence a `root:apache` e o usuário `opc` não tem permissão de leitura. Confira com `sudo`:

```bash
sudo ls /opt/sei
```

Devem aparecer `bin`, `config`, `scripts`, `temp` e `web`.

## 6. Arquivos de configuração

### 6.2 Repositório de arquivos externos

Crie o diretório do repositório e aplique as permissões:

```bash
sudo mkdir -p /sei-nfs/dados
sudo chown apache:apache /sei-nfs/dados
sudo chmod 770 /sei-nfs/dados
```

Se o SELinux estiver em `Enforcing`, aplique também o contexto:

```bash
sudo chcon -R -t httpd_sys_rw_content_t /sei-nfs/dados
```

### 6.3 Configure o SIP

Faça backup do arquivo e abra-o para edição:

```bash
sudo cp /opt/sip/config/ConfiguracaoSip.php /opt/sip/config/ConfiguracaoSip.php.bak
sudo vi /opt/sip/config/ConfiguracaoSip.php
```

| Chave | Valor |
| --- | --- |
| `SIP` / `URL` | `http://127.0.0.1/sip` |
| `SIP` / `Producao` | `true` |
| `SessaoSip` / `SiglaOrgaoSistema` | `LAB` |
| `SessaoSip` / `PaginaLogin` | `http://127.0.0.1/sip/login.php` |
| `SessaoSip` / `SipWsdl` | `http://127.0.0.1/sip/controlador_ws.php?servico=sip` |
| `SessaoSip` / `https` | `false` |
| `BancoSip` | Servidor `127.0.0.1`, Porta `3306`, Banco `sip`, Usuario `sip_user`, Senha a sua, Tipo `MySql` |
| `BancoAuditoriaSip` | os mesmos valores do `BancoSip` |
| `CacheSip` | Servidor `127.0.0.1`, Porta `11211` |
| `InfraMail` | `Tipo` `2`, servidor SMTP do lab (ou deixe como está por enquanto) |

#### Exemplo de configuração

```
sudo tee /opt/sip/config/ConfiguracaoSip.php > /dev/null <<'EOF'
<?php

class ConfiguracaoSip extends InfraConfiguracao
{
private static $instance = null;
public static function getInstance()
{
    if (ConfiguracaoSip::$instance == null) {
        ConfiguracaoSip::$instance = new ConfiguracaoSip();
    }
    return ConfiguracaoSip::$instance;
}

public function getArrConfiguracoes()
{
    return array(
        'Sip' => array(
            'URL' => '<http://127.0.0.1/sip>',
            'Producao' => true,
            'NumLoginSemCaptcha' => 3,
            'TempoLimiteValidacaoLogin' => 60,
            'Modulos' => array(
                //'ABCExemploIntegracao' => 'abc/exemplo',
            ),
        ),

        'PaginaSip' => array(
            'NomeSistema' => 'Sistema Unico de Processo Eletronico em Rede',
            'NomeSistemaComplemento' => '',
        ),

        'SessaoSip' => array(
            'SiglaOrgaoSistema' => 'LAB',
            'SiglaSistema' => 'SIP',
            'PaginaLogin' => '<http://127.0.0.1/sip/login.php>',
            'SipWsdl' => '<http://127.0.0.1/sip/controlador_ws.php?servico=sip>',
            'ChaveAcesso' => 'd27791b894028d9e7fa34887ad6f0c9a2c559cccda5f64f4e108e3573d5db862b66fb933',
            'https' => false,
        ),

        'BancoSip' => array(
            'Servidor' => '127.0.0.1',
            'Porta' => '3306',
            'Banco' => 'sip',
            'Usuario' => 'sip_user',
            'Senha' => 'SENHA_SIP_AQUI',
            'Tipo' => 'MySql',
            'PesquisaCaseInsensitive' => false,
        ),

        //'BancoAuditoriaSip' => array(
        //    'Servidor' => '[Servidor BD]',
        //    'Porta' => '',
        //    'Banco' => '',
        //    'Usuario' => '',
        //    'Senha' => '',
        //    'Tipo' => '',
        //    'PesquisaCaseInsensitive' => false,
        //),

        'CacheSip' => array(
            'Servidor' => '127.0.0.1',
            'Porta' => '11211'
        ),

        'InfraMail' => array(
            'Tipo' => '1',
            'Servidor' => '127.0.0.1',
            'Porta' => '25',
            'Codificacao' => '8bit',
            'Autenticar' => false,
            'Usuario' => '',
            'Senha' => '',
            'Seguranca' => '',
            'MaxDestinatarios' => 999,
            'MaxTamAnexosMb' => 999,
            'Protegido' => '',
        ),
    );
}
}
EOF
```

#### 6.3.1 Configure a senha, as permissões e o backup

sudo sed -i "s|SENHA_SIP_AQUI|${SENHA_SIP}|" /opt/sip/config/ConfiguracaoSip.php
sudo chown root:apache /opt/sip/config/ConfiguracaoSip.php
sudo chmod 640 /opt/sip/config/ConfiguracaoSip.php
sudo cp /opt/sip/config/ConfiguracaoSip.php /opt/sip/config/ConfiguracaoSip.php.bak
unset SENHA_SIP

#### 6.3.2 Valide a configuração SIP

sudo grep -c -E 'SENHA_SIP_AQUI|deixe o valor|SUA_SENHA' /opt/sip/config/ConfiguracaoSip.php
sudo head -3 /opt/sip/config/ConfiguracaoSip.php
php -v

- O `grep` deve dar `0` (nenhum resto do arquivo errado nem do placeholder).
- O `head` deve começar com `<?php`.
- O `php -v` precisa mostrar `PHP 7.3.x`. Se aparecer `command not found`, o PHP ainda não foi instalado e isso trava o sistema inteiro. Nesse caso volte ao passo do EPEL e do PHP antes de continuar.

### 6.4 Configure o SEI

Faça backup do arquivo e abra-o para edição:

```bash
sudo cp /opt/sei/config/ConfiguracaoSEI.php /opt/sei/config/ConfiguracaoSEI.php.bak
sudo vi /opt/sei/config/ConfiguracaoSEI.php
```

| Chave | Valor |
| --- | --- |
| `SEI` / `URL` | `http://127.0.0.1/sei` |
| `SEI` / `Producao` | `true` |
| `SEI` / `RepositorioArquivos` | `/sei-nfs/dados` |
| `SessaoSEI` / `SiglaOrgaoSistema` | `LAB` |
| `SessaoSEI` / `PaginaLogin` | `http://127.0.0.1/sip/login.php` |
| `SessaoSEI` / `SipWsdl` | `http://127.0.0.1/sip/controlador_ws.php?servico=sip` |
| `SessaoSEI` / `https` | `false` |
| `BancoSEI` e `BancoAuditoriaSEI` | Servidor `127.0.0.1`, Porta `3306`, Banco `sei`, Usuario `sei_user`, Senha a sua, Tipo `MySql` |
| `CacheSEI` | Servidor `127.0.0.1`, Porta `11211` |

#### Exemplo de configuração

```
sudo tee /opt/sei/config/ConfiguracaoSEI.php > /dev/null <<'EOF'
<?php

class ConfiguracaoSEI extends InfraConfiguracao
{
private static $instance = null;
public static function getInstance()
{
    if (ConfiguracaoSEI::$instance == null) {
        ConfiguracaoSEI::$instance = new ConfiguracaoSEI();
    }
    return ConfiguracaoSEI::$instance;
}

public function getArrConfiguracoes()
{
    return array(
        'SEI' => array(
            'URL' => '<http://127.0.0.1/sei>',
            'Producao' => true,
            'DigitosDocumento' => 7,
            'PermitirAcessoLocalPdf' => '',
            'NumLoginUsuarioExternoSemCaptcha' => 3,
            'TamSenhaUsuarioExterno' => 8,
            'DebugWebServices' => 0,
            'RepositorioArquivos' => '/sei-nfs/dados',
            'Modulos' => array(
                //'MdAbcExemploIntegracao' => 'abc/exemplo',
            ),
        ),

        'PaginaSEI' => array(
            'NomeSistema' => 'Sistema Unico de Processo Eletronico em Rede',
            'NomeSistemaComplemento' => '',
            'LogoMenu' => '',
            'OrgaoTopoJanela' => 'S',
        ),

        'SessaoSEI' => array(
            'SiglaOrgaoSistema' => 'LAB',
            'SiglaSistema' => 'SEI',
            'PaginaLogin' => '<http://127.0.0.1/sip/login.php>',
            'SipWsdl' => '<http://127.0.0.1/sip/controlador_ws.php?servico=sip>',
            'ChaveAcesso' => '7babf862e12bd48f3101075c399040303d94a493c7ce9306470f719bb453e0428c6135dc',
            'https' => false,
        ),

        'BancoSEI' => array(
            'Servidor' => '127.0.0.1',
            'Porta' => '3306',
            'Banco' => 'sei',
            'Usuario' => 'sei_user',
            'Senha' => 'SENHA_SEI_AQUI',
            'Tipo' => 'MySql',
            'PesquisaCaseInsensitive' => false,
        ),

        //'BancoAuditoriaSEI' => array(
        //    'Servidor' => '[servidor BD]',
        //    'Porta' => '',
        //    'Banco' => '',
        //    'Usuario' => '',
        //    'Senha' => '',
        //    'Tipo' => '',
        //    'PesquisaCaseInsensitive' => false,
        //),

        'CacheSEI' => array(
            'Servidor' => '127.0.0.1',
            'Porta' => '11211'
        ),

        'Solr' => array(
            'Servidor' => '<http://127.0.0.1:8983/solr>',
            'CoreProtocolos' => 'sei-protocolos',
            'CoreBasesConhecimento' => 'sei-bases-conhecimento',
            'CorePublicacoes' => 'sei-publicacoes',
            'TempoCommitProtocolos' => 300,
            'TempoCommitBasesConhecimento' => 60,
            'TempoCommitPublicacoes' => 60,
        ),

        //'JODConverter' => array(
        //    'Servidor' => 'http://[Servidor JodConverter]/converter/service'
        //),

        'Federacao' => array(
            'Habilitado' => false,
        ),

        'InfraMail' => array(
            'Tipo' => '1',
            'Servidor' => '127.0.0.1',
            'Porta' => '25',
            'Codificacao' => '8bit',
            'Autenticar' => false,
            'Usuario' => '',
            'Senha' => '',
            'Seguranca' => 'TLS',
            'MaxDestinatarios' => 999,
            'MaxTamAnexosMb' => 999,
            'Protegido' => '',
        ),

        'Limites' => array(
            'Nivel1TempoSeg' => 60,
            'Nivel1MemoriaMb' => 256,
            'Nivel2TempoSeg' => 600,
            'Nivel2MemoriaMb' => 2048,
            'Nivel3TempoSeg' => 0,
            'Nivel3MemoriaMb' => 4096,
        ),
    );
}
}
EOF
```

#### 6.4.1 Configure a senha, as permissões e o backup

sudo sed -i "s|SENHA_SEI_AQUI|${SENHA_SEI}|" /opt/sei/config/ConfiguracaoSEI.php
unset SENHA_SEI
sudo chown root:apache /opt/sei/config/ConfiguracaoSEI.php
sudo chmod 640 /opt/sei/config/ConfiguracaoSEI.php
sudo cp /opt/sei/config/ConfiguracaoSEI.php /opt/sei/config/ConfiguracaoSEI.php.bak

### 6.5 Valide a sintaxe e o Memcached

```bash
sudo php -l /opt/sip/config/ConfiguracaoSip.php
sudo php -l /opt/sei/config/ConfiguracaoSEI.php
sudo systemctl status memcached --no-pager
```

Os dois `php -l` devem responder `No syntax errors detected`. Se o `php` ainda não estiver instalado, volte ao passo do PHP primeiro, porque sem ele o SEI não roda. Me diga se o PHP 7.3 já foi instalado.

### 6.6 Pendências antes do teste

```bash
sudo mkdir -p /sei-nfs/dados
sudo chown apache:apache /sei-nfs/dados
sudo chmod 770 /sei-nfs/dados
getenforce
php -i | grep -E '^(short_open_tag|default_charset|include_path|post_max_size|upload_max_filesize)'
sudo systemctl status memcached --no-pager | head -5
```

- Se o `getenforce` responder `Enforcing`, rode também `sudo chcon -R -t httpd_sys_rw_content_t /sei-nfs/dados` e os `setsebool` do passo 4.3.
- Se o `php -i` não mostrar `short_open_tag => On`, `default_charset => ISO-8859-1` e `include_path => .:/opt/infra/infra_php`, falta aplicar os `sed` do `php.ini` (passo 4.2).
- Se o `memcached` não estiver `active (running)`: `sudo dnf install -y memcached && sudo systemctl enable --now memcached`.

Depois disso: `sudo systemctl restart httpd` e abra `http://152.70.210.189/sip` com usuário e senha `teste`.

### 6.7 Teste inicial

```bash
sudo systemctl restart httpd
```

Abra `http://127.0.0.1/sip` no navegador e teste o login. Se houver erro, consulte `Infra/Log` do SIP e `/var/log/httpd/error_log` (a seção 22 do manual lista as mensagens mais comuns).

Sobre o Solr: deixei o bloco apontando para `127.0.0.1`, que ainda não existe neste servidor. O sistema abre e funciona, mas a pesquisa em conteúdo de documentos não vai funcionar, e podem aparecer erros de indexação no `Infra/Log`. Se isso atrapalhar, a gente instala o Solr depois (seção 21 do manual). O `JODConverter` ficou comentado porque é opcional.

## Solr (seção 21 do manual)

O Solr habilita a pesquisa no conteúdo dos documentos. Como a VM tem 7,47 GB de RAM, compartilhados pelo MySQL, Apache e PHP, configurei 2 GB de heap em vez dos 6 GB indicados no manual.

### Instalação

```bash
sudo dnf install -y java-1.8.0-openjdk
java -version
sudo useradd solr
cd /tmp
curl -LO https://archive.apache.org/dist/lucene/solr/8.2.0/solr-8.2.0.tgz
sudo cp -r /opt/sei/config/solr/. /tmp/
ls /tmp
```

Confirme que aparecem `log4j.properties`, `sei-solr-8.2.0.sh`, `sei-cores-8.2.0` e `solr.service`. Antes de executar o script como root, leia-o e confira se ele instala o Solr em `/opt/solr` e cria os índices em `/dados`:

```bash
sudo cat /tmp/sei-solr-8.2.0.sh
```

Se o conteúdo estiver correto, execute o script e confira os diretórios criados:

```bash
sudo bash /tmp/sei-solr-8.2.0.sh
sudo tree -d /dados 2>/dev/null || sudo ls /dados
```

Devem existir os diretórios `sei-protocolos`, `sei-bases-conhecimento` e `sei-publicacoes`.

### Configuração e inicialização

Defina o heap máximo como 2 GB e configure o Solr para escutar apenas em `127.0.0.1`, pois o SEI está na mesma máquina:

```bash
sudo sed -i 's|^#\?SOLR_JAVA_MEM=.*|SOLR_JAVA_MEM="-Xms512m -Xmx2048m"|' /opt/solr/bin/solr.in.sh
echo 'SOLR_JETTY_HOST="127.0.0.1"' | sudo tee -a /opt/solr/bin/solr.in.sh
sudo systemctl daemon-reload
sudo systemctl enable --now solr
sleep 20
curl -s 'http://127.0.0.1:8983/solr/admin/cores?action=STATUS' | head -c 300
```

O endereço `127.0.0.1` limita o acesso ao Solr à própria VM. O comando de consulta deve retornar o status dos cores.

### Indexação inicial

Para indexar os dados que já estão no banco, execute:

```bash
sudo php -c /etc/php.ini /opt/sei/scripts/indexacao_protocolos_completa.php
sudo php -c /etc/php.ini /opt/sei/scripts/indexacao_publicacoes.php
sudo php -c /etc/php.ini /opt/sei/scripts/indexacao_bases_conhecimento.php
```

![image.png](image%201.png)

![image 1.png](image_1.png)

### 7.1 Resultado dos testes e diagnóstico do endereço

As capturas de tela mostram o servidor respondendo localmente, mas também evidenciam um problema ao acessar o IP público a partir da própria VM.

- O `curl` e o `lynx` em `127.0.0.1/sip/` recebem `302` e são redirecionados para `http://152.70.210.189/sip/login.php?...`. Esse é o comportamento certo do SIP: ele manda o navegador para o endereço que está configurado.
- Depois disso, de **dentro da VM**, falha com `No route to host` ao tentar alcançar o próprio IP público (`152.70.210.189`). Isso é comum em VM de nuvem, que não consegue se chamar pelo IP público. O teste do `lynx` falhou por isso, e não por defeito no SIP.

Isso não prova que o navegador do seu computador também falha. Mas importa para outra coisa: o SEI e o SIP **chamam um ao outro por web service** (`SipWsdl` e o `web_service` do banco), e essas chamadas saem da VM. Com o IP público nelas, o login do SEI vai dar “Erro acessando arquivo WSDL” (problema 3 da seção 22). Então já vale corrigir.

### 7.2 Use `127.0.0.1` nas chamadas entre serviços

Só `SipWsdl` (nos dois arquivos) e o `web_service` do SEI no banco mudam. As páginas que o navegador abre continuam com o IP público.

sudo sed -i "s|'SipWsdl' => 'http://127.0.0.1|'SipWsdl' => 'http://127.0.0.1|" /opt/sip/config/ConfiguracaoSip.php /opt/sei/config/ConfiguracaoSEI.php

mysql -u root -p <<'EOF'
USE sip;
update sistema set web_service='[http://127.0.0.1/sei/controlador_ws.php?servico=sip](http://127.0.0.1/sei/controlador_ws.php?servico=sip)' where sigla='SEI';
SELECT sigla, pagina_inicial, web_service FROM sistema;
EOF

sudo php -l /opt/sip/config/ConfiguracaoSip.php
sudo php -l /opt/sei/config/ConfiguracaoSEI.php
sudo systemctl restart memcached httpd

O `SELECT` deve mostrar `pagina_inicial` ainda com o IP público e `web_service` do SEI com `127.0.0.1`.

### 7.3 Teste a tela de login pela VM

Como o redirecionamento leva ao IP público, teste direto a página de login (as aspas são necessárias por causa do `&`):

curl -s -o /dev/null -w '%{http_code}\n' '[http://127.0.0.1/sip/login.php?sigla_orgao_sistema=LAB&sigla_sistema=SIP](http://127.0.0.1/sip/login.php?sigla_orgao_sistema=LAB&sigla_sistema=SIP)'
curl -s '[http://127.0.0.1/sip/login.php?sigla_orgao_sistema=LAB&sigla_sistema=SIP](http://127.0.0.1/sip/login.php?sigla_orgao_sistema=LAB&sigla_sistema=SIP)' | sed 's/<[[1]](>)*>/ /g' | grep -v '^\s*$' | head -15

O primeiro comando deve retornar `200` e o segundo deve mostrar a tela de login. Se retornar `500`, consulte o log:

```bash
sudo tail -5 /var/log/php-fpm/www-error.log
```

### 7.4 Confira se o WSDL aponta para um endereço alcançável

curl -s '[http://127.0.0.1/sip/controlador_ws.php?servico=sip](http://127.0.0.1/sip/controlador_ws.php?servico=sip)' | grep -i -o 'location="[[2]](")*"' | head -3

- Se aparecer `127.0.0.1`, está tudo certo.
- Se aparecer `152.70.210.189`, o SEI vai tentar chamar o SIP por esse IP, que a VM não alcança. Nesse caso, dá para fazer a VM responder ao próprio IP público localmente:

sudo ip addr add 152.70.210.189/32 dev lo
curl -s -o /dev/null -w '%{http_code}\n' [http://152.70.210.189/sip/login.php](http://152.70.210.189/sip/login.php)

### 7.5 Acesso local por tunelamento

Para acessar o [localhost](http://localhost), utilize o tunelamento com as configurações mostradas na imagem abaixo.

![image 2.png](image_2.png)
