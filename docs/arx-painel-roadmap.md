# Arx Painel — Roadmap de Robustez, Segurança e Funcionalidades

O Arx Painel é a interface web de administração do Arx OS. Este documento
descreve a arquitetura de segurança do projeto, o conjunto de módulos
implementados e o estado atual de cada um.

O princípio central do projeto é a **separação de privilégios**: a
interface web nunca executa ações administrativas diretamente — toda
operação sensível passa por um componente privilegiado separado, com
validação e uma lista fechada de ações permitidas.

```text
Navegador
   ↓  HTTPS
nginx
   ↓  socket Unix
webui (sem privilégio, usuário painel-web)
   ↓  socket Unix
helper (root)
   ↓  ações explicitamente permitidas (whitelist)
systemd / apt / arquivos de configuração / ferramentas do sistema
```

Os dois processos (`webui` e `helper`) rodam confinados por **perfis
AppArmor em modo enforce** — mesmo que um dos dois seja comprometido, o
kernel limita o que ele pode tocar no sistema de arquivos, independente
de qualquer bug de validação na aplicação.

## Estado geral do projeto

| Fase | Escopo | Progresso |
|---|---|---|
| 1 — Segurança e estabilidade | HTTPS, sessões, auditoria, rollback, health checks | ✅ 10/10 |
| 2 — Administração básica | Serviços, logs, usuários, grupos, diagnóstico, monitoramento, discos | ✅ 7/7 |
| 3 — Infraestrutura | DNS, DHCP, firewall, NTP, Samba AD | ✅ completos, com ressalvas pontuais abaixo |
| 4 — Operações profissionais | Jobs assíncronos, backup completo, alertas, relatórios | ✅ 6/6 |
| 5 — Recursos avançados | RBAC, MFA, WebAuthn, API completa, plugins | ✅ 5/6 (item 6 adiado conscientemente) |

---

## 1. Princípios arquiteturais

### 1.1 Separação de privilégios

O `webui` roda sem privilégios administrativos e nunca invoca
`subprocess`, `os.system`, shell ou comandos do sistema diretamente. Toda
ação privilegiada existe explicitamente numa tabela de ações permitidas
no `helper`, que valida rigorosamente cada parâmetro recebido — mesmo
quando o `webui` já os validou antes. Comandos administrativos usam
sempre caminhos absolutos, e `shell=True` é evitado sistematicamente.

### 1.2 Fail-safe

Alterações de rede e firewall (as que podem cortar o acesso remoto do
próprio administrador) seguem um modelo de **confirmação com janela de
tempo**: a mudança é aplicada, e se não for confirmada em **15 segundos**,
reverte sozinha automaticamente. Outras alterações críticas (DNS, DHCP)
seguem o fluxo `validar → backup → testar sintaxe/configuração → aplicar
→ verificar`, com reversão pro backup em caso de falha na verificação.

### 1.3 Auditoria

Ações administrativas são registradas com usuário, data/hora, ação,
parâmetros não sensíveis e resultado — nunca senhas, tokens ou outros
segredos.

---

## 2. Dashboard

Visão resumida do estado do servidor: CPU, RAM, disco, uptime, kernel,
hostname, IP principal, e os eventos mais recentes de auditoria e
alertas. Timestamps em hora local, formato legível.

## 3. Sistema

Hostname, kernel, arquitetura, uptime legível, uso de CPU (load average),
memória e disco — em tempo real via `system.resource_status`.

## 4. Monitoramento

Métricas históricas com retenção configurável, mais um snapshot ao vivo.
Inclui também estatísticas dedicadas de consultas DNS (`dns_stats`),
agregadas a partir do log de consultas do BIND.

## 5. Serviços systemd

Consulta de status de serviços críticos do próprio painel e da
infraestrutura gerenciada (nginx, BIND, DHCP, Samba, chrony), com health
check dedicado que classifica cada um em 6 níveis de gravidade (ver seção
"Sistema de gravidade" abaixo).

## 6. Logs

Auditoria completa e paginada (`/logs`), com usuário, IP, ação e
resultado de cada operação feita através do painel.

## 7. Rede

Interfaces (listagem, ligar/desligar), IP estático, rotas, e VLANs
completas — criação (`create_vlan`), listagem e remoção, com validação de
interface pai, VLAN ID, endereço e prefixo. Mudanças de rede usam o
modelo de confirmação com janela de 15s.

## 8. Diagnóstico de rede

`ping`, `traceroute`, `dns_lookup` (consulta por tipo de registro) e
`port_check` (teste de porta TCP), todos com validação de host/alvo antes
de executar — nunca interpolação direta em shell.

## 9. Firewall

Gerenciamento do `nftables`: regras (adicionar, remover), status,
persistência entre reboots, inicialização de regras padrão. Regras
estruturais (loopback, conexões estabelecidas) são protegidas — não
removíveis pela interface. Mudanças passam pelo modelo de confirmação com
janela de 15s (reversão automática se não confirmado).

## 10. DNS (BIND9)

Gerenciamento de zonas, registros, modo raw de configuração, estatísticas
de consultas (`dns_stats`), controle de query log, e um apelido de zona
protegido (`arx.os`). Validação de configuração antes de aplicar.

## 11. DHCP

CRUD completo de sub-redes, reservas e opções, com modo raw de
configuração e listagem de leases ativos.

## 12. NTP / Chrony

Status de sincronização real (fonte, stratum, offset) via `chronyc`,
separado do status "configurado" — um servidor configurado pode não
estar sincronizando de fato. **Tolerância de boot**: nos primeiros 5
minutos após ligar, NTP ainda não sincronizado é tratado como estado
`info` (esperado), não `critico` — evita alerta/notificação falso a cada
reinicialização.

## 13. Usuários locais

Gerenciamento de contas locais e, via Samba AD, contas de domínio —
criação, desativação, ativação, geração de senha provisória compatível
com política do AD. Senhas nunca em texto puro nos logs.

## 14. Grupos

Criação, remoção e gerenciamento de membros de grupos locais.

## 15. Samba AD

Módulo mais extenso do projeto (34 funções): usuários, grupos,
computadores, OUs, zonas DNS integradas ao domínio, status do domínio,
movimentação de usuário entre OUs. Provisionamento inicial e operações
demoradas rodam como job assíncrono acompanhável.

## 16. Atualizações

Verificação e aplicação de atualizações via job assíncrono, com detecção
de necessidade de reboot — nunca reinicia sozinho sem ação explícita do
administrador.

## 17. Backup

Backup e restauração completos das configurações geridas pelo painel.
Restaurar exige confirmação digitada, e cria automaticamente um backup de
segurança do estado atual antes de sobrescrever.

## 18. Rollback

Dois mecanismos, conforme o risco:
- **Mudança de rede/firewall**: aplica e reverte sozinho se não
  confirmado em 15 segundos (evita perda de acesso remoto)
- **Mudança de config com validação prévia** (DNS, DHCP): valida sintaxe
  antes de aplicar; se a verificação pós-aplicação falhar, reverte pro
  backup automático

## 19. Auditoria

Ver seção 1.3 — completo.

## 20. Alertas

Sistema de 6 níveis de gravidade (não 3): `ok`, `info`, `atencao`,
`alto`, `critico`, `emergencia`. Monitora serviços críticos do painel,
espaço em disco (gradual), firewall, DHCP, DNS, NTP (com tolerância de
boot), Samba AD, reinício pendente e modo manutenção. Histórico completo
de transições de estado, com notificação por e-mail e/ou webhook
(canais independentes — um falhar não impede o outro) disparada em
thread de background pra nunca travar a interface.

## 21. Autenticação

Senhas como hash (`werkzeug.security`), proteção CSRF em toda rota POST
(exceto API por token, que usa outro modelo de autenticação), sessões com
`HttpOnly`, bloqueio temporário após tentativas falhas de login.

## 22. HTTPS

Sempre via HTTPS (porta 9006), certificado autoassinado por padrão (ou
confiável, se o admin configurar), headers de segurança (HSTS,
X-Content-Type-Options, X-Frame-Options, CSP restrito). A aplicação
Flask/Gunicorn nunca fica exposta diretamente — todo tráfego passa por
nginx via socket Unix.

## 23. Gestão de sessões

Sessão expira ao reiniciar o serviço webui por padrão (chave de sessão
aleatória, a menos que `ARX_PAINEL_SECRET_KEY` seja configurada como
variável de ambiente persistente).

## 24. Controle de acesso (RBAC)

**Dois papéis** — `admin` (acesso completo) e `leitura` (visualização,
bloqueado de qualquer ação que altera o sistema). A validação acontece
tanto no `webui` (defesa em profundidade — botões de ação viram
"Somente leitura" na interface) quanto, de forma autoritativa, no
`helper` — nenhuma ação de escrita passa sem essa checagem central,
independente da interface.

## 25. Segurança do helper

Socket Unix, grupo dedicado, ações explícitas numa whitelist central,
validação de parâmetros, timeouts. Roda confinado por perfil AppArmor
próprio, em modo enforce.

## 26. Jobs assíncronos

Operações demoradas (provisionamento Samba, atualizações, restauração)
rodam como jobs rastreáveis, com status consultável via
`GET /api/jobs/<id>`.

## 27. Tratamento de erros

Erros do helper chegam ao `webui` como mensagem estruturada, nunca como
stack trace exposta ao navegador.

## 28. Configuração e segredos

Configuração e segredos em `/etc/painel/` e `/etc/arx/`, dados
persistentes como arquivos JSON (ver seção 30). Segredos sensíveis
(senha SMTP, credencial de serviço) criptografados em repouso (Fernet),
nunca em texto puro no disco. Tokens de API armazenados como hash
SHA-256 — o texto puro só existe uma vez, no momento da criação, e não é
recuperável depois.

## 29. Empacotamento Debian

Pacote `.deb` com `postinst`/`prerm`/`postrm`, venv Python próprio para
o `webui` com dependências via pip, perfis AppArmor instalados e
recarregados automaticamente a cada atualização.

## 30. Armazenamento de dados

**Arquivos JSON**, não banco de dados relacional — decisão consciente
pelo volume de dado do projeto (configuração, auditoria, alertas, tokens,
admins). Escrita protegida por lock de arquivo para lidar com
concorrência entre o `webui` e os timers periódicos do `helper`.

## 31. API administrativa

**Endpoint genérico único**, não REST por domínio: `POST
/api/v1/executar`, autenticado por token (`Authorization: Bearer
arxapi_...`), recebe `{"acao": "modulo.funcao", "params": {...}}` e
expõe as mesmas ações que o `webui` usa internamente — decisão
consciente pra reaproveitar as 130+ ações já existentes (e o RBAC que já
vem com elas) em vez de duplicar rota por rota. O token herda o papel de
quem criou (`admin` ou `leitura`).

Complementado por `GET /api/v1/eventos` — dois modos na mesma rota:
polling (`?desde=<timestamp>`) e streaming via Server-Sent Events
(conexão persistente, usada pelo app companion Arx Sentinela). O
`gunicorn` roda com worker `gevent` especificamente pra suportar
conexões SSE de longa duração sem travar os workers disponíveis para o
resto do painel.

Tokens são criados e revogados pela própria interface web
(`/api-tokens`), com recuperação assistida — um admin pode revogar o
token de outra conta (`/admins/<usuario>/tokens`) se necessário.

## 32. Interface

Interface escura, consistente, com confirmação explícita (às vezes
digitada) antes de qualquer ação potencialmente destrutiva. Ícones em
estilo linha (stroke, viewBox 24×24), mesma convenção do conjunto Lucide
Icons.

## 33. Saúde dos módulos

`GET /saude` e a ação `health.get_status` consolidam o estado de todos
os serviços monitorados num único lugar, com os 6 níveis de gravidade.

## 34. Diagnóstico consolidado

`/doctor` — visão de suporte consolidada: saúde dos serviços, modo dos
perfis AppArmor, atualizações pendentes, validade do certificado HTTPS.

## 35. Transações administrativas

Ver seções 1.2 e 18.

## 36. Testes automatizados

Sem suíte de testes formal (pytest) no repositório — a validação de cada
mudança acontece manualmente a cada iteração (testes funcionais diretos:
sintaxe, colisão de rota, fluxo HTTP completo, casos de erro) antes de
empacotar. **Gap real**: não há testes automatizados que rodem sozinhos
em CI.

## 37. Observabilidade interna

Página de Saúde e `arx doctor` cobrem isso — não existe um sistema
separado de métricas de observabilidade do próprio painel (tempo de
resposta, taxa de erro) além do que já está coberto pelo health check.

---

## Sistema de gravidade (6 níveis)

Usado em todo o sistema de alertas/saúde — não são 3 níveis (info,
atenção, crítico), são 6:

| Nível | Cor | Significado |
|---|---|---|
| `ok` | verde | normal |
| `info` | azul | informativo, sem ação corretiva (ex: NTP ainda sincronizando após boot) |
| `atencao` | amarelo | risco moderado |
| `alto` | laranja | risco alto, requer preparação |
| `critico` | vermelho | falha grave, ação urgente |
| `emergencia` | roxo | situação gravíssima |

---

## Estado atual (atualizado 2026-09-23)

Legenda: ✅ feito e testado · ⚠️ parcial/ressalva · ⏸ adiado conscientemente

### Fase 1 — Segurança e estabilidade — 10/10 completa

| # | Item | Status |
|---|---|---|
| 1 | HTTPS | ✅ |
| 2 | SECRET_KEY externa | ✅ (opcional — sessão não sobrevive a restart sem ela configurada) |
| 3 | Rate limiting / bloqueio de força bruta | ✅ |
| 4 | Sessões seguras | ✅ |
| 5 | Auditoria | ✅ |
| 6 | Tratamento de erros | ✅ |
| 7 | Validação centralizada no helper | ✅ |
| 8 | Backup antes de mudanças | ✅ |
| 9 | Rollback | ✅ (dois modelos, ver seção 18) |
| 10 | Health checks | ✅ |

### Fase 2 — Administração básica — 7/7

| # | Item | Status |
|---|---|---|
| 11 | Serviços systemd | ✅ |
| 12 | Logs | ✅ |
| 13 | Usuários | ✅ locais + domínio |
| 14 | Grupos | ✅ |
| 15 | Diagnóstico de rede | ✅ ping, traceroute, dns lookup, port check |
| 16 | Monitoramento | ✅ histórico + snapshot ao vivo + estatísticas DNS |
| 17 | Discos/SMART | ✅ partições, espaço, LUKS, SMART (com fallback gracioso em disco virtual sem suporte) |

### Fase 3 — Infraestrutura — completos, com ressalvas pontuais

| # | Item | Status |
|---|---|---|
| 18 | DNS | ✅ zonas, registros, raw, estatísticas, zona protegida |
| 19 | DHCP | ✅ CRUD completo, raw, leases |
| 20 | Firewall | ✅ regras, persistência, estruturais protegidas — ⚠️ sem zonas/templates avançados |
| 21 | NTP | ✅ status real + configurado, com tolerância de boot |
| 22 | Samba AD | ✅ módulo mais extenso (34 funções) — usuários, grupos, OUs, DNS integrado |

### Fase 4 — Operações profissionais — 6/6

| # | Item | Status |
|---|---|---|
| 23 | Jobs assíncronos | ✅ |
| 24 | Backup completo | ✅ |
| 25 | Restauração | ✅ com confirmação digitada + backup de segurança automático |
| 26 | Alertas | ✅ 6 níveis, notificação e-mail/webhook |
| 27 | Relatórios de diagnóstico | ✅ |
| 28 | Histórico de métricas | ✅ |

### Fase 5 — Recursos avançados — 5/6

| # | Item | Status |
|---|---|---|
| 29 | RBAC | ✅ (2 papéis: admin, leitura) |
| 30 | MFA (TOTP) | ✅ com QR code |
| 31 | WebAuthn/passkeys | ✅ chave física ou biometria, escolha entre TOTP e WebAuthn no login |
| 32 | API administrativa completa | ✅ endpoint genérico por token + streaming SSE |
| 33 | Plugins/módulos | ⏸ adiado conscientemente pra pós-1.0 — arquitetura atual (webui↔helper por ação nomeada) já é a fundação certa pra isso depois, decisão documentada |

### Extras não previstos no roadmap original

- **AppArmor em modo enforce** (não só complain) nos dois processos, validado contra múltiplas rodadas de uso real antes da migração
- **Notificações** por e-mail e webhook, canais independentes
- **Config portátil** — exportar/importar configuração
- **Modo manutenção** — suprime alertas durante trabalho planejado
- **Certificados** — status de validade do HTTPS
- **App companion (Arx Sentinela)** — Flutter, recebe status/eventos via API + SSE, sem depender de FCM/ntfy

---

## Critérios para considerar o Arx Painel "profissional"

- [x] Nenhuma execução arbitrária de comandos pelo navegador
- [x] Helper separado e mínimo, confinado por AppArmor
- [x] Validação no webui e no helper
- [x] HTTPS
- [x] CSRF
- [x] Rate limiting / bloqueio de força bruta
- [x] Sessões seguras
- [x] Auditoria
- [x] Logs
- [x] Backup completo
- [x] Rollback (dois modelos, conforme o risco)
- [x] Jobs assíncronos
- [x] Health checks
- [x] Tratamento de erros
- [x] RBAC
- [ ] Testes automatizados (validação manual a cada mudança, sem suíte formal em CI)
- [x] Empacotamento Debian
- [x] Segredos fora do código, criptografados em repouso
- [x] Nenhuma alteração crítica sem validação
- [x] Proteção contra perda de acesso remoto (janela de confirmação de 15s)

## Regra de ouro para novas funcionalidades

Antes de implementar qualquer recurso novo, o projeto avalia:

1. Essa ação precisa de root?
2. Se sim, ela realmente precisa passar pelo helper?
3. O parâmetro pode ser validado por whitelist?
4. É possível fazer a operação sem shell?
5. Existe risco de perder acesso ao servidor?
6. Existe backup antes da alteração?
7. Existe validação antes de aplicar?
8. Existe rollback?
9. A operação precisa ser assíncrona?
10. A ação precisa aparecer na auditoria?
11. O erro pode ser apresentado de forma segura?
12. O recurso continua funcionando se o navegador for fechado?

Qualquer resposta que indique risco leva o projeto a priorizar segurança
e reversibilidade antes da conveniência.

## Arquitetura final

```text
                         ┌─────────────────────┐
                         │      Navegador      │
                         └──────────┬──────────┘
                                    │ HTTPS (porta 9006)
                                    ▼
                         ┌─────────────────────┐
                         │        nginx        │
                         │   CSP / headers de  │
                         │      segurança      │
                         └──────────┬──────────┘
                                    │ Unix socket
                                    ▼
                    ┌──────────────────────────────┐
                    │          Arx WebUI           │
                    │      RBAC / CSRF / sessão /  │
                    │ rate-limit / gunicorn+gevent │
                    │     confinado por AppArmor   │
                    └──────────────┬───────────────┘
                                   │ Unix socket
                                   ▼
                    ┌──────────────────────────────┐
                    │          Arx Helper          │
                    │             root             │
                    │       whitelist de ações     │
                    │    validação / autorização   │
                    │    confinado por AppArmor    │
                    └──────────────┬───────────────┘
                                   │
             ┌─────────────────────┼──────────────────────┐
             ▼                     ▼                      ▼
         systemd                  apt              arquivos/config
             │                     │                      │
             ▼                     ▼                      ▼
        Serviços              Atualizações          DNS/DHCP/Firewall/etc.
```

O objetivo do Arx Painel não é ser apenas uma interface gráfica para
comandos Linux, mas uma camada administrativa com privilégio mínimo,
validação, auditoria, reversibilidade e diagnóstico integrado.
