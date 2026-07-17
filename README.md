# Manual de Referência: Monitoramento SNMPv1 com Zabbix e Grafana

## Cenário do Laboratório

| Componente | Descrição |
|------------|-----------|
| **Servidor Gerente** | Ubuntu (VirtualBox) com **Zabbix + Grafana** |
| **IP do Gerente** | `192.168.1.121` |
| **Servidor Agente** | Ubuntu Server (**kaua**) |
| **IP do Agente** | `192.168.1.120` |
| **Protocolo de Coleta** | SNMPv1 |
| **Community** | `public` |

---

# Fase 1: Configuração do Agente SNMP
**Máquina:** `kaua (192.168.1.120)`

O objetivo desta fase é configurar o servidor alvo para escutar requisições SNMP na porta **UDP 161** e permitir leitura completa utilizando a comunidade **public**.

## 1. Instalar o daemon do SNMP

```bash
sudo apt update
sudo apt install snmpd -y
```

---

## 2. Configurar o arquivo de escuta e segurança

Abra o arquivo de configuração:

```bash
sudo nano /etc/snmp/snmpd.conf
```

### Liberar escuta na rede

Comente a linha de loopback local e adicione a escuta global:

```text
#agentaddress 127.0.0.1,[::1]
agentaddress udp:161
```

### Definir a comunidade de leitura

Localize a configuração da comunidade `public` e remova a diretiva `-V systemonly`, permitindo acesso a todas as métricas (CPU, memória, rede etc.):

```text
rocommunity public default
```

> **Observação de segurança:** Em ambiente de produção, substitua `default` pelo IP do servidor gerente (`192.168.1.121`).

---

## 3. Reiniciar e habilitar o serviço

```bash
sudo systemctl restart snmpd
sudo systemctl enable snmpd
```

---

## 4. Testar localmente (Opcional)

```bash
snmpwalk -v 1 -c public 127.0.0.1 sysName
```

---

# Fase 2: Instalação e Configuração do Gerente
**Servidor Zabbix:** `192.168.1.121`

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

# Fase 3: Configuração Web e Cadastro do Host

## 1. Finalizar a instalação via navegador

Acesse:

```
http://192.168.1.121/zabbix
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
| Host name | `kaua` |
| Template | Linux by SNMP (ou Generic by SNMP) |
| Host Group | Linux servers |

---

### Adicionar Interface SNMP

Clique em:

```
Add → SNMP
```

Configure:

| Campo | Valor |
|--------|--------|
| IP | `192.168.1.120` |
| SNMP Version | SNMPv1 |
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
kaua
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
http://192.168.1.121:3000
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

## 4. Criar um Dashboard

Acesse:

```
Dashboards
    └── New Dashboard
```

Clique no botão:

```
+ Add Visualization
```

Selecione o **Data Source Zabbix**.

Na consulta (**Query A**) configure:

| Campo | Valor |
|--------|--------|
| Group | Linux servers |
| Host | kaua |
| Item | CPU utilization, Available memory ou Interfaces de Rede |

---

## 5. Ajustar período de visualização

No canto superior direito altere o intervalo para:

- Last 5 minutes

ou

- Last 15 minutes

Assim será possível visualizar as primeiras coletas e acompanhar a geração dos gráficos em tempo real.

---

# Resultado Esperado

Ao final deste procedimento você terá:

- ✅ SNMPv1 configurado no servidor Ubuntu (`kaua`);
- ✅ Zabbix Server monitorando o host via SNMP;
- ✅ Banco MySQL configurado e integrado ao Zabbix;
- ✅ Interface Web do Zabbix funcional;
- ✅ Grafana conectado ao Zabbix;
- ✅ Dashboards exibindo métricas como:
  - Utilização de CPU;
  - Memória disponível;
  - Interfaces de rede;
  - Demais métricas disponibilizadas pelo SNMP.

