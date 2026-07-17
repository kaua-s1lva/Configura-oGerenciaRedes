# Manual de Referência: Monitoramento SNMPv1 com Zabbix e Grafana

## Cenário do Laboratório

| Componente | Descrição |
|------------|-----------|
| **Servidor Gerente** | Ubuntu (VirtualBox) com **Zabbix + Grafana** |
| **Servidores Agentes** | SW-01, SW-02, SW-03 e RT-01 |
| **IP dos Agentes** | `10.10.20.110`, `10.10.20.111`, `10.10.20.112`, `10.10.20.100` |
| **Protocolo de Coleta** | SNMPv2 |
| **Community** | `public` |

---

# Fase 1: Instalação e Configuração do Gerente
Esta fase prepara o banco de dados MySQL, instala o servidor Zabbix e realiza sua configuração.

---

## 1. Adicionar o repositório oficial do Zabbix

> Exemplo para **Ubuntu 24.04**

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb

sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb

sudo apt update
```

---

## 2. Instalar os pacotes necessários

```bash
sudo apt install mysql-server \
zabbix-server-mysql \
zabbix-frontend-php \
zabbix-apache-conf \
zabbix-sql-scripts \
zabbix-agent -y
```

---

## 3. Iniciar e habilitar o MySQL

```bash
sudo systemctl start mysql
sudo systemctl enable mysql
```

---

## 4. Criar o banco de dados

Entre no MySQL:

```bash
sudo mysql
```

Execute:

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '123';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

SET GLOBAL log_bin_trust_function_creators = 1;

QUIT;
```

---

## 5. Importar o esquema do banco

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Quando solicitado, informe a senha:

```text
123
```

> Aguarde aproximadamente **1 a 2 minutos** até o término da importação.

---

## 6. Restaurar a configuração do MySQL

```bash
sudo mysql -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

---

## 7. Configurar a senha do banco no Zabbix

Abra o arquivo:

```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Localize:

```text
DBPassword=
```

Altere para:

```text
DBPassword=123
```

---

## 8. Reiniciar os serviços

```bash
sudo systemctl restart zabbix-server zabbix-agent apache2

sudo systemctl enable zabbix-server zabbix-agent apache2
```

---

# Fase 2: Configuração Web e Cadastro do Host

## 1. Finalizar a instalação via navegador

Acesse:

```
localhost/zabbix
```

Durante o assistente:

- Avance normalmente pelas etapas.
- Na configuração do banco informe a senha:

```text
123
```

---

## 2. Login inicial

**Usuário**

```text
Admin
```

**Senha**

```text
zabbix
```

---

## 3. Cadastrar o host

Acesse:

```
Data Collection → Hosts → Create Host
```

Configure:

| Campo | Valor |
|-------|-------|
| Host name | `SW-01` |
| Template | Cisco IOS by SNMP |
| Host Group | Switch |

---

### Adicionar Interface SNMP

Clique em:

```
Add → SNMP
```

Configure:

| Campo | Valor |
|--------|--------|
| IP | `10.10.20.110` |
| SNMP Version | SNMPv2 |
| Community | `public` |

Clique em **Add** para salvar.

---

### Verificar comunicação

Após alguns instantes, a etiqueta **SNMP** deverá aparecer em **verde** na lista de hosts.

---

## 4. Visualizar dados

Acesse:

```
Monitoring → Latest Data
```

Em **Hosts**, selecione:

```
SW-01
```

Clique em **Apply** para visualizar as métricas coletadas.

---

# Fase 4: Integração Visual com o Grafana

## 1. Instalar o Grafana

```bash
sudo mkdir -p /etc/apt/keyrings/

wget -q -O - https://apt.grafana.com/gpg.key | \
gpg --dearmor | \
sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt update

sudo apt install grafana -y

sudo systemctl daemon-reload

sudo systemctl start grafana-server

sudo systemctl enable grafana-server
```

---

## 2. Ativar o plugin do Zabbix

Acesse:

```
localhost:3000
```

Login padrão:

| Campo | Valor |
|--------|--------|
| Usuário | admin |
| Senha | admin |

Depois:

```
Administration
    └── Plugins and Data
          └── Plugins
```

Pesquise por:

```
Zabbix
```

Autor:

```
Alexander Zobnin
```

Clique em:

```
Enable
```

---

## 3. Configurar a fonte de dados

Acesse:

```
Connections
    └── Data Sources
          └── Add Data Source
```

Selecione:

```
Zabbix
```

Configure:

**URL**

```text
http://localhost/zabbix/api_jsonrpc.php
```

Na seção **Zabbix API Auth**:

- Enable
- Usuário: **Admin**
- Senha: **senha do painel do Zabbix**

Clique em:

```
Save & Test
```

Uma mensagem verde deverá confirmar que a conexão foi estabelecida.

---

# Resultado Esperado

Ao final deste procedimento você terá:

- ✅ SNMPv2 configurado no servidor Ubuntu;
- ✅ Zabbix Server monitorando o host via SNMP;
- ✅ Banco MySQL configurado e integrado ao Zabbix;
- ✅ Interface Web do Zabbix funcional;
- ✅ Grafana conectado ao Zabbix;
- ✅ Dashboards exibindo métricas como:
  - Utilização de CPU;
  - Memória disponível;
  - Interfaces de rede;
  - Demais métricas disponibilizadas pelo SNMP.

