# Lab Blue Team: Detectando Alterações Maliciosas com FIM no Wazuh
## Objetivo
Atualmente, os ataques de cibercriminosos estão cada vez mais sofisticados, utilizando táticas avançadas para alterar arquivos críticos do sistema, pastas, registros e dados de endpoint. Neste laboratório, foquei em detectar a criação, modificação e exclusão de arquivos em diretórios sensíveis utilizando o módulo Syscheck do Wazuh. O objetivo é entender os dados que permitem diferenciar uma alteração legítima de uma atividade potencialmente maliciosa.

## Topologia
Servidor Linux (Ubuntu + Wazuh Agent) → Wazuh Manager

## 0. Baseline: O estado do sistema antes do monitoramento
Antes de aplicar qualquer regra de monitoramento, é fundamental estabelecer a linha de base (baseline) do nosso agente Ubuntu. Documentei o estado exato dos arquivos que serão nossos alvos de teste:

Verificando o arquivo que será criado:
O diretório /tmp está limpo em relação ao nosso arquivo de teste.

Verificando o arquivo que será modificado:
Capturei o hash MD5 original e as últimas linhas do arquivo de configuração do SSH (/etc/ssh/sshd_config) antes de qualquer alteração.

<img width="652" height="270" alt="image" src="https://github.com/user-attachments/assets/e56d6880-c7cd-4149-8eb7-30a8db01b252" />

Com o estado original documentado, iniciei a configuração do monitoramento.

## O que está acontecendo
O módulo Syscheck do agente guarda um hash (checksum) dos arquivos e diretórios configurados para monitoramento. Em modo realtime, ele detecta a mudança no momento em que ela acontece (via inotify no Linux), recalcula o hash e dispara um alerta contendo o hash antigo, o novo, o caminho exato e o tipo de evento (added, modified ou deleted).

---

## 1. Configuração do escopo de monitoramento
No Wazuh Manager, configurei os diretórios sensíveis que serão monitorados. Utilizei a seguinte configuração via agent.conf compartilhado (recomendado, pois escala para múltiplos agentes):
<img width="696" height="264" alt="image" src="https://github.com/user-attachments/assets/0229a460-f456-4972-9730-1f1528a9bf72" />


## 2. Forçar sincronização no agente
Para não depender do ciclo automático (padrão de 12h), forcei uma nova varredura:
```bash
sudo systemctl restart wazuh-agent
```
---

## 3. Gerar as três alterações de teste

**3.1 — Criação de arquivo (added):**
```bash
echo "arquivo suspeito" > /tmp/arquivo_suspeito.txt
```

**3.2 — Modificação de arquivo existente (modified):**
```bash
echo "# teste FIM" | sudo tee -a /etc/ssh/sshd_config
```

**3.3 — Exclusão de arquivo (deleted):**
```bash
rm /tmp/arquivo_suspeito.txt
```
<img width="769" height="83" alt="image" src="https://github.com/user-attachments/assets/7fa7388f-1bae-4170-9f66-f28e9f34eaaf" />

> ⚠️ Evite alterar `/etc/passwd` diretamente em ambiente real — nesse lab, se optar por testar nele, sempre faça backup antes (`sudo cp /etc/passwd /etc/passwd.bak`) e restaure ao final.> 
---

## 4. Observar a detecção no Wazuh
<img width="1548" height="589" alt="image" src="https://github.com/user-attachments/assets/cd47b085-c527-4365-9688-e72a06278c43" />

Os 3 eventos distintos,

```
syscheck.event: added
syscheck.path: /tmp/arquivo_suspeito.txt
```
<img width="664" height="949" alt="image" src="https://github.com/user-attachments/assets/1c3c92cf-edb5-48ba-b179-b41f92e8d031" />

```
syscheck.event: modified
syscheck.path: /etc/ssh/sshd_config
syscheck.md5_before: [hash antigo]
syscheck.md5_after: [hash novo]
```
<img width="669" height="957" alt="image" src="https://github.com/user-attachments/assets/9d4ae2e1-d09c-40e3-ad33-57a7768ae1b7" />

<img width="865" height="924" alt="image" src="https://github.com/user-attachments/assets/a75a9b48-0121-4630-87ec-77bee20edb06" />


```
syscheck.event: deleted
syscheck.path: /tmp/arquivo_suspeito.txt
```
<img width="670" height="982" alt="image" src="https://github.com/user-attachments/assets/291c52e3-c32c-4fb6-9974-f932e2d07ceb" />

---

## 5. Painel dedicado
No Wazuh temos um painel dedicado em Endpoint Security → File Integrity Monitoring. Esse painel agrega os eventos de FIM por agente e por tipo, sendo extremamente útil para uma visão consolidada em operações de SOC, sem a necessidade de montar a query manualmente no Discover.
<img width="1915" height="858" alt="image" src="https://github.com/user-attachments/assets/525c0c53-6108-4e07-af9d-942f0236eae3" />

<img width="1913" height="533" alt="image" src="https://github.com/user-attachments/assets/55c474e6-d10e-4472-874f-efd26c080826" />

## Conclusão
O FIM é a base da detecção de manipulação não autorizada de arquivos — desde um ransomware alterando arquivos em massa até um atacante plantando um webshell ou modificando configurações de serviços (como o sshd_config) para garantir persistência. Entender o dado bruto (hash antes/depois, caminho, tipo de evento, usuário responsável) é o que permite ao analista diferenciar uma mudança legítima de uma maliciosa. O alerta sozinho não conta a história completa sem esse contexto investigativo.

## Aprendizados
O modo realtime="yes" (via inotify) permite detecção quase instantânea, o que é essencial para diretórios críticos como /etc e /etc/ssh, onde o ciclo padrão de 12 horas deixaria uma janela de exposição muito ampla.

A comparação criptográfica (hash) antes/depois é a evidência mais confiável para confirmar que o conteúdo mudou, descartando falsos positivos gerados por simples alterações de metadados.

Arquivos de configuração de serviços (como o sshd_config) são alvos táticos clássicos. Uma alteração aqui frequentemente indica uma tentativa de enfraquecer controles de acesso (ex: permitir login direto de root, mudar a porta, ou permitir autenticação por senha quando deveria ser exclusivamente por chave).

