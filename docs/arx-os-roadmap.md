# Arx OS — Roadmap Geral

Roadmap técnico da distro **Arx OS**: decisões de arquitetura, fases de
build (Fase 0 a 5) e o estado real de cada componente — kernel, base do
sistema, modo Servidor, modo Cliente/Desktop, instalador e empacotamento.
Para o detalhamento específico do painel web de administração, ver
[`arx-painel-roadmap.md`](arx-painel-roadmap.md).

## Sumário

- [Status geral](#status-geral-atualizado-2026-09-23)
- [Changelog resumido do arx-painel](#changelog-resumido-do-arx-painel-01x--0361)
- [Visão geral do projeto](#visão-geral-do-projeto)
- [Fase 0 — Decisões de arquitetura](#fase-0--decisões-de-arquitetura-antes-de-compilar-qualquer-coisa)
- [Fase 1 — Base do sistema](#fase-1--base-do-sistema-comum-aos-dois-modos)
- [Fase 2 — Modo Servidor](#fase-2--modo-servidor)
- [Fase 3 — Modo Cliente/Desktop](#fase-3--modo-clientedesktop)
- [Perfil "Terminal Público"](#perfil-terminal-público--pcs-de-hallpesquisa-cpu-fraca-8gb-ram-ssd)
- [Qual kernel usar](#qual-kernel-usar)
- [Fase 4 — Instalador gráfico](#fase-4--instalador-gráfico)
- [Fase 5 — Empacotamento e distribuição](#fase-5--empacotamento-e-distribuição)
- [O que recompilar vs. o que vem do Debian](#lista-o-que-recompilar-vs-o-que-vem-do-debian-main)
- [Repositório apt próprio](#repositório-apt-próprio--arquitetura)
- [Autenticação — AD vs. conta local](#autenticação--ad-vs-conta-local)
- [Painel web — arquitetura de privilégios](#painel-web--arquitetura-de-privilégios-importante-pro-apparmor)
- [O que usamos como base — resumo](#o-que-usamos-como-base--resumo)
- [Ordem de implementação prática](#ordem-de-implementação-prática)
- [CLI `arx` e loja de aplicativos](#cli-arx-e-loja-de-aplicativos-decidido-em-2026-09-12-não-previsto-originalmente)
- [Grupos de máquinas](#grupos-de-máquinas--software-por-grupo-ex-lab-1--lab-2)

---

## Status geral (atualizado 2026-09-23)

| Fase | Escopo | Status |
|---|---|---|
| 0 — Decisões de arquitetura | apt/dpkg, kernel, AppArmor, Calamares, canais de pacote | ✅ completa |
| 1 — Base do sistema | kernel 6.18.45, LUKS, rede de fábrica, repositório apt | ✅ completa |
| 2 — Modo Servidor | `arx-painel` v0.36.1 | ✅ completa (5/6 da Fase 5 de robustez do painel — ver changelog) |
| 3 — Modo Cliente/Desktop | KDE Plasma, NetworkManager, Terminal Público/Quiosque | ❌ não iniciada |
| 4 — Instalador (Calamares) | tela Servidor/Cliente, firmware opt-in | ❌ não iniciado — depende da Fase 3 |
| 5 — Empacotamento/ISO | geração de ISO, build reprodutível | ❌ não iniciado — depende da Fase 3/4 |
| `arx-cli` | wrapper sobre apt, publicado em `main` | ✅ básico funcionando · loja de apps depende da Fase 3 |
| Arx Sentinela *(novo, fora do roadmap original)* | app companion (Flutter) | API do painel pronta · app ainda não iniciado |

Detalhes de cada item:

<details>
<summary>Ver detalhamento completo do status geral</summary>

- **Fase 0 (decisões de arquitetura)**: ✅ completa — apt/dpkg, kernel deblobbed+hardened, AppArmor, Calamares, canais stable/edge/unstable definidos
- **Fase 1 (base do sistema)**: ✅ completa — kernel 6.18.45 (server+desktop) compilado/testado, LUKS testado no servidor, rede de fábrica configurada (DoT via systemd-resolved), repositório apt próprio rodando de verdade (`main`+`unstable`+`stable`+`edge`)
- **Fase 2 (Modo Servidor)**: ✅ **completa e validada em produção real** — `arx-painel` (v0.36.1) com Rede/Firewall/DHCP/DNS/NTP/Domínio(Samba AD)/Usuários/Grupos/Logs/Auditoria/Health checks/Diagnóstico/Jobs assíncronos/Backup+Restauração/Alertas/Relatórios/Notificações, todos testados em VM real. **Roadmap de robustez do painel tem as Fases 1-4 completas (28/28) e a Fase 5 em 5/6** — RBAC, MFA (TOTP), WebAuthn/passkeys, API completa (por token, com streaming de eventos), e os dois perfis **AppArmor do próprio painel em modo enforce** (validado contra múltiplas rodadas de log real antes da migração). Só falta Plugins, adiado conscientemente pra pós-1.0 — ver changelog abaixo
- **Fase 3 (Modo Cliente/Desktop)**: ❌ **não iniciada** — KDE Plasma, NetworkManager, app de configurações Qt, perfis Terminal Público/Quiosque continuam só no papel
- **Instalador (Calamares)**: ❌ não iniciado — depende da Fase 3
- **`arx-cli`**: CLI wrapper sobre apt (`/usr/local/bin/arx`), com autocomplete, publicado no `main`. Arquitetura de "loja de aplicativos" decidida, mas a integração desktop (sssd/PAM+AD) que ela depende ainda não existe
- **Repositório apt**: quatro componentes publicados (`main`, `unstable`, `stable`, `edge`). Kernel 6.18.45 promovido pro `stable`. `edge` ainda vazio (esperando um kernel mainline 7.2.x pra publicar ali)
- **Arx Sentinela** *(novo, fora do roadmap original)*: app companion (Flutter, mobile+desktop) em desenvolvimento, recebe status/alertas do painel em tempo real via API por token + streaming (SSE), sem depender de FCM/ntfy. Endpoint `/api/v1/eventos` já implementado e testado no painel; o app em si ainda não foi iniciado

</details>

---

## Changelog resumido do arx-painel (0.1.x → 0.36.1)

Histórico completo, versão por versão, em `painel/README.md` (esse aqui é só o resumo dos marcos grandes).

- **0.1–0.4**: construção inicial do painel — módulos core (Rede, Firewall, DHCP, DNS, NTP, Domínio, Usuários), fundação do design system (sidebar, cards, botões)
- **0.5–0.6**: **Fase 1 do roadmap de robustez completa** — HTTPS, SECRET_KEY externa, rate limiting, sessões seguras, auditoria, tratamento de erros, validação centralizada, backup/rollback generalizado, health checks
- **0.7–0.8**: refinamento de DHCP/DNS, vários bugs reais corrigidos (DHCPv6, sub-redes de exemplo do Debian aparecendo como ativas, postinst não reiniciando serviço no upgrade), componentes de design system (abas, modal de confirmação, chips removíveis)
- **0.9**: Dashboard e página de Serviços redesenhados (referência visual do usuário), apelido fixo `arx.os` pra acessar o painel sem decorar IP
- **0.10–0.11**: **Fase 2 completa** (diagnóstico de rede, SMART, histórico de métricas) — e início da **Fase 3** com o item mais crítico: rollback automático do firewall (dead-man's switch — aplica e desfaz sozinho em 15s se não confirmar), motivado por dois lockouts reais durante o desenvolvimento
- **0.12–0.13**: rollback automático generalizado (extraído pra utilitário compartilhado, aplicado também em Rede — IP fixo e VLAN). **Fase 3 completa**: contadores de pacote/byte no firewall, DNS com AAAA/PTR/zona reversa, DHCP com leases ativos de verdade, NTP com status real de sincronização. Bug crítico corrigido: rollback automático só vivia em memória — se o helper reiniciasse no meio da janela de 15s, a mudança arriscada ficava aplicada pra sempre; corrigido com persistência em disco e recuperação automática no boot (testado com stop completo real do processo, não só restart)
- **0.14–0.15**: **Fase 4 completa** — jobs assíncronos (ações demoradas rodam em background), backup completo do sistema + restauração (com backup de segurança automático antes de restaurar, upload de backup externo), alertas (histórico persistido de degradações de saúde), relatório de diagnóstico exportável
- **0.16**: sistema de gravidade revisado pra 6 níveis (verde/azul/âmbar/laranja/vermelho/roxo — antes só tinha 3), corrigindo classificações reais (firewall sem nenhuma proteção era "alerta"/amarelo, virou "crítico"/vermelho), sino de alertas no menu com contagem e cor da pior gravidade ativa
- **0.17–0.29**: início da **Fase 5** — RBAC (papéis `admin`/`leitura`, checagem central no helper + defesa em profundidade na interface), MFA via TOTP (RFC 6238 implementado direto, QR code, recuperação assistida por outro admin), notificações por e-mail e webhook (canais independentes, thread de background)
- **0.30–0.33**: **WebAuthn/passkeys** (chave física ou biometria, escolha entre TOTP e WebAuthn no login) — bugs reais de proxy corrigidos no caminho (nginx repassando porta errada pro Flask, exigência de certificado confiável documentada). **API completa por token**: endpoint genérico único (`POST /api/v1/executar`, ação nomeada + parâmetros) em vez de REST por domínio — decisão consciente pra reaproveitar as 130+ ações já existentes; token herda o papel de quem criou, revogável (inclusive por outro admin, em nome de alguém)
- **0.34**: **migração dos dois perfis AppArmor pra modo enforce** — depois de várias rodadas de revisão de log real de produção (bugs reais encontrados: caminho de symlink não batendo com o caminho resolvido pelo kernel em `/sbin` vs `/usr/sbin`, binários faltando na whitelist, arquivo de grupos locais sem cobertura). Corrida de boot do `isc-dhcp-server` corrigida de vez (dependência mais forte em `systemd-networkd-wait-online` + checagem ativa de IP). Tolerância de boot pro alerta de NTP (nível "info" nos primeiros minutos, evita notificação falsa a cada reinicialização)
- **0.35–0.36**: página de espera 100% autossuficiente ao reiniciar/desligar (sem depender de arquivo externo, que não sobrevive à corrida real entre o navegador buscar CSS/JS e o servidor começar a desligar). **Endpoint `/api/v1/eventos`** (polling + streaming via Server-Sent Events) pro app companion **Arx Sentinela** — gunicorn migrado pro worker `gevent` (necessário pra conexão persistente não travar os workers disponíveis pro resto do painel)

**Com isso, o roadmap de robustez do painel tem as Fases 1 a 4 completas (28/28) e a Fase 5 em 5/6** — só falta Plugins/módulos, **adiado conscientemente** pra pós-1.0 (arquitetura atual do painel — webui sem privilégio ↔ socket ↔ helper privilegiado por ação nomeada — já é a fundação certa pra isso depois, decisão documentada).

---

## Visão geral do projeto

Uma distro construída a partir do Linux From Scratch, com **dois modos de instalação mutuamente exclusivos**:

- **Modo Servidor** → gerenciado via painel web
- **Modo Cliente/Desktop** → gerenciado via interface gráfica nativa

Ambos compartilham a mesma base (toolchain, kernel hardened, políticas de segurança), mas divergem no conjunto de pacotes e na camada de gerenciamento.

---

## Fase 0 — Decisões de arquitetura (antes de compilar qualquer coisa)

Decisões já travadas:

1. **Gerenciador de pacotes: dpkg/apt** (Debian). Ganha em suporte de comunidade e ferramental pronto, mas traz uma implicação prática: tudo que for compilado via processo LFS precisa ser empacotado como `.deb` (com `dpkg-deb`, `equivs` ou scripts de build próprios que gerem o `control`/dependências corretos) pra entrar no apt normalmente. Duas formas de tocar isso:
   - **Empacotar manualmente** cada pacote compilado do jeito LFS como `.deb` — mais trabalho por pacote, mas te dá controle total sobre flags de compilação/hardening em cada um
   - **Puxar pacotes já prontos dos repositórios Debian** (main, sem non-free/contrib) pra tudo que não precisa de hardening customizado, e reservar o processo LFS/recompilação só pro que realmente importa (kernel, toolchain, componentes de segurança) — isso reduz muito o trabalho e mantém controle total sobre a curadoria de pacotes e o hardening
   - Segunda abordagem escolhida como ponto de partida — puro LFS pra absolutamente tudo, mais empacotamento `.deb` manual de cada peça, é um volume de trabalho que só faz sentido se hardening de cada pacote for realmente o objetivo

2. **Baseline do kernel/sistema: 100% livre, sem blobs proprietários.** Isso significa:
   - Kernel deblobbed nos moldes do `linux-libre` (remove firmware não-livre embutido) — dá pra aplicar os scripts de deblob do linux-libre num kernel vanilla + patches de hardening (`CONFIG_HARDENED_USERCOPY`, `CONFIG_STACKPROTECTOR_STRONG`, `CONFIG_FORTIFY_SOURCE`, `linux-hardened` patchset)
   - Repositório apt restrito a `main` (sem `contrib`/`non-free`/`non-free-firmware`)
   - Consequência prática: sem firmware não-livre, hardware que depende dele (algumas placas Wi-Fi, GPUs proprietárias) não funciona — é a mesma limitação que Trisquel/Parabola/PureOS enfrentam. Registrado como nota de compatibilidade de hardware, não como impedimento
   - MAC/sandboxing: AppArmor tende a ser mais simples de manter num projeto solo que SELinux, e já tem integração razoável no ecossistema Debian

3. **Instalador: Live ISO com Calamares**, customizado com a tela de seleção Servidor/Cliente. Como o gerenciador de pacotes agora é apt, o módulo de pacotes do Calamares (`packagechooser`) já tem suporte nativo a apt — outro ganho de tempo por ter escolhido dpkg/apt.

---

## Fase 1 — Base do sistema (comum aos dois modos)

- Build LFS/BLFS padrão (toolchain, glibc, etc.), com scripts de build versionados (não fazer manual toda vez — automatize com algo tipo Jhalfs adaptado)
- **Kernel: 7.2 (mainline atual) ou 6.18 (longterm/LTS)** — ver seção "Qual kernel usar" abaixo, decisão pendente
- Firewall default-deny (nftables — mais moderno que iptables pra base nova) já na imagem base
- Política de privacidade por padrão: sem telemetria, DNS via DoT/DoH configurado de fábrica, logs mínimos com opt-in pra logs verbosos
- **Rede configurada de fábrica** (DHCP como fallback sensato) — observado durante os testes que um rootfs cru de `debootstrap` não sobe rede sozinho; a base final do Arx OS precisa resolver isso, não pode depender de configuração manual pós-boot
- Disco criptografado por padrão **só no modo Servidor** (LUKS) — no Desktop fica de fora por padrão, já que o usuário não tem a senha de disco pra digitar no boot (PCs de uso coletivo/público precisam ligar sozinhos, sem ninguém presente digitando senha); no servidor faz sentido como camada extra, já que é gerenciado com acesso físico controlado. **Nota técnica confirmada em teste**: o instalador precisa gravar `GRUB_ENABLE_CRYPTODISK=y` em `/etc/default/grub` sempre que o disco for criptografado, senão o GRUB cai num shell interativo em vez de mostrar o menu (bug fácil de esquecer, não é automático)
- Repositório apt próprio (restrito a `main`, sem non-free) com os pacotes compilados via LFS/hardening + espelho dos pacotes Debian `main` usados como base

## Fase 2 — Modo Servidor

- Painel web (Flask, aproveitando a experiência que você já tem do [[filtro-dns-academico]] — gunicorn, systemd services)
- Módulos do painel:
  - Rede: interfaces, **VLANs (802.1Q)**, roteamento, **IP fixo configurável** (servidor não deve depender só de DHCP — precisa da opção de IP estático, já que outras máquinas apontam pra ele como referência fixa, ex: o próprio repositório apt)
  - Firewall (nftables) via UI
  - Usuários/serviços do sistema
  - Atualizações (integrado ao apt, com o repositório próprio configurado)
  - **Domínio** (Samba AD DC — provisionamento, usuários/grupos/OUs)
  - **Grupos de máquinas** — conjuntos de software por grupo (ex: "Lab 1", "Lab 2"), instalação disparada do servidor
  - **Repositório local** (decidido em 2026-09-14, **revisado em 2026-09-15 pra modo cache**, `arx-painel` 0.18.0): cada servidor roda um proxy de cache (`apt-cacher-ng`) — só baixa e guarda o que uma máquina de domínio pede de verdade, em vez de espelhar o Debian inteiro antecipado. **Mudança de arquitetura em cima da decisão original** (que era espelho completo via `aptly`, testado em produção): hardware fraco (2 núcleos) travava tentando espelhar tudo de uma vez, mesmo filtrado. Bônus real da mudança: elimina de vez a complexidade de chave GPG que o espelho completo exigia (o cache não verifica assinatura, quem faz isso continua sendo o `apt` de cada cliente). Cobre Debian de fábrica + repositório central do Arx configurado à parte. **Ainda em aberto**: como um cliente NÃO-domínio (standalone) descobre pra onde apontar — decisão a fechar quando a Fase 3 (Desktop) começar
- Pacotes mínimos: sem ambiente gráfico nenhum instalado por padrão

## Fase 3 — Modo Cliente/Desktop

- **Ambiente gráfico: KDE Plasma** (decidido — customização e facilidade de uso)
- **Rede: NetworkManager**, não `systemd-networkd` — decisão específica do perfil desktop, diferente do servidor. Motivo: `systemd-networkd` é headless/declarativo por design, sem nenhuma história de interface gráfica; o Plasma já tem integração nativa e madura com NetworkManager (applet de Wi-Fi, VPN, tudo clicável), que é o que um usuário comum de desktop espera
- App de configurações nativo em Qt/QML (consistente com o Plasma, em vez de GTK)
- Sandboxing de apps (bubblewrap/firejail) como padrão para reduzir superfície de ataque
- **Ajustes de privacidade específicos do KDE** (o Plasma vem com alguns serviços que conflitam com o objetivo do projeto se deixados no padrão):
  - **Baloo** (indexador de arquivos) — desligado por padrão, ligar via painel de configurações só se o usuário quiser busca rápida
  - **KUserFeedback** (telemetria opcional de algumas apps KDE) — desligado por padrão, sem exceção
  - **KWallet** — mantido, mas integrado com o login (PAM), pra não pedir senha duplicada
  - Rever se `drkonqi` (crash reporter) deve enviar relatório pra algum lugar externo — por padrão, não

### Firmware e drivers de terceiros (contrib/non-free/non-free-firmware) — decidido 2026-09-14

Kernel continua **deblobbed** (linux-libre, sem nenhum firmware não-livre embutido) — isso não muda, é parte da identidade "Secure · Private · Free" da distro. O que muda: os componentes `contrib`, `non-free` e `non-free-firmware` do Debian passam a existir como **opção**, não como padrão.

- **Por padrão, em qualquer instalação** (Servidor, Desktop, Quiosque): só `main`/`stable` ativos — sistema 100% livre, do jeito que já era
- **No instalador (Calamares)**: uma tela de opção — se o usuário marcar "permitir drivers de terceiros" (AMD/Intel/NVIDIA, firmware de Wi-Fi/Bluetooth, etc), o sistema já sobe com `contrib`/`non-free`/`non-free-firmware` habilitados no `sources.list`
- **Depois de instalado**: mesma escolha disponível a qualquer momento pela interface gráfica (app de configurações Qt/QML do modo Desktop) — liga ou desliga esses componentes facilmente, pra quando trocar de placa de vídeo/Wi-Fi ou simplesmente mudar de ideia
- Motivação real: hardware variado (principalmente no perfil "Terminal Público", que mira PCs velhos/diversos) muitas vezes só funciona direito (Wi-Fi, aceleração de vídeo) com firmware não-livre. Negar essa opção forçaria hardware real a não funcionar, sem ganho de privacidade de verdade (o firmware roda no próprio chip, não no kernel do sistema)
- Isso é expansão de escolha, não desvio de filosofia — controle final sempre com quem instala/usa a máquina, nunca embutido "por debaixo dos panos"
- Módulo do painel de admin (Servidor) não precisa dessa opção — é uma decisão do modo Desktop/Quiosque, onde tem interface gráfica e hardware variado de verdade

## Perfil "Terminal Público" — PCs de hall/pesquisa (CPU fraca, 8GB+ RAM, SSD)

Terceiro perfil dentro do modo Cliente/Desktop, pensado especificamente pra máquinas de uso coletivo (hall, sala de pesquisa) — hardware tipo Athlon II X2: CPU é o gargalo real, RAM (8GB+) e disco (SSD) sobram. Isso inverte a prioridade de otimização de "economizar memória" pra "economizar ciclo de CPU":

- **LXQt** em vez de KDE Plasma completo — mesmo ecossistema Qt (o app de configurações nativo serve pros dois perfis), mas sem compositor pesado nem os daemons de fundo do Plasma
- **Compositor/efeitos desligados por padrão** (sombra, transparência, animação) — é onde CPU fraca mais sofre visivelmente
- Mesma política de privacidade do perfil desktop completo (nada de indexação de arquivo tipo Baloo rodando à toa — em CPU fraca isso é ainda mais crítico do que só privacidade)
- **Sessão efêmera**: `$HOME` roda sobre um overlay que reseta a cada logout — como sobra RAM, dá pra usar `tmpfs` como camada de escrita do overlay (rápido, e não desgasta o SSD com escrita constante de sessões descartáveis). Todo download, histórico de navegador, config alterada por um usuário some pro próximo
- Encaixa como um valor do módulo **Grupos de máquinas** já projetado: esses PCs formam um grupo próprio (ex: "Hall — Terminal Público"), com esse perfil leve + sessão efêmera, enquanto laboratórios/salas de aula usam o Plasma completo

### Modo Quiosque (opcional, por grupo)

Toggle adicional no mesmo grupo "Hall — Terminal Público", no painel: liga uma sessão trancada em vez do LXQt normal.

- **Sessão dedicada** (`/usr/share/xsessions/arx-kiosk.desktop`): sem painel, sem área de trabalho — só um gerenciador de janela mínimo (Openbox, sem atalho nenhum configurado) rodando o navegador em `--kiosk` (tela cheia, sem barra de endereço/menu)
- **Login automático** com uma conta local dedicada e sem privilégio (não usa AD aqui — evita expor credencial de domínio numa máquina de acesso público)
- **Navegação aberta, não travada num site só** — já que o uso real é "pesquisa e acesso à internet" (assumindo isso como padrão; se quiser travar numa lista de sites específicos, é só trocar a política do navegador depois)
- **Política do navegador travada**: sem instalar extensão, sem abrir DevTools, sem alterar configuração — via `policies.json` (Firefox) ou política equivalente (Chromium)
- **Sem fuga pro sistema**: troca de TTY (Ctrl+Alt+F2, etc.) desabilitada na sessão, sem acesso a terminal ou gerenciador de arquivo
- **Saída pra manutenção**: um atalho de teclado específico (ex: Ctrl+Alt+Shift+Esc) pede senha de administrador local e derruba pra uma sessão normal — pra equipe técnica destravar sem precisar reinstalar nada
- Reaproveita a **sessão efêmera em `tmpfs`** já definida no perfil "Terminal Público" — cache/histórico/download do navegador somem a cada logout de qualquer forma, então o quiosque nem precisa de lógica de limpeza própria

## Qual kernel usar

Duas opções reais em setembro de 2026:

| | 7.2 (mainline) | 6.18 (longterm/LTS) |
|---|---|---|
| Suporte | Só até a próxima versão +3 meses (ciclo de ~2 meses) | Mantido por anos |
| Hardware recente | Melhor (suporte mais novo, ex: Apple M3 parcial, USB4STREAM) | Um pouco atrás |
| Maturidade dos patches de hardening (`linux-hardened`) e do deblob (`linux-libre`) | Podem estar atrasados em relação à versão exata — em setembro de 2026, o `linux-libre` estável era baseado em 7.0.x | Mais provável já ter cobertura completa |
| Esforço de manutenção pro projeto | Alto — troca de kernel a cada ~2 meses pra acompanhar "mainline" | Baixo — atualiza dentro da mesma série por bastante tempo |

**Decidido:** `stable` = 6.18 LTS, `edge` = 7.2 (mainline atual sem LTS), `unstable` = versões em desenvolvimento tipo 7.3-rc1. Confirmado que o linux-libre já publica `6.18-gnu` deblobbado (saiu junto com o lançamento do 6.18 mainline), então a base de deblob pro kernel stable já está madura.



## Fase 4 — Instalador gráfico

- Calamares customizado (ou equivalente) com:
  - Tela de seleção **Servidor vs Cliente** (exclusiva)
  - Seleção de pacotes opcionais dentro do modo escolhido
  - **Modo Cliente**: opção de habilitar `contrib`/`non-free`/`non-free-firmware` (drivers AMD/Intel/NVIDIA, firmware Wi-Fi/Bluetooth) — desligado por padrão, mesma escolha disponível depois pela interface gráfica (ver seção na Fase 3)
  - Particionamento com LUKS pré-configurado como padrão recomendado

## Fase 5 — Empacotamento e distribuição

- Build reprodutível (scripts versionados, não builds manuais "artesanais")
- Geração de ISO (xorriso ou o próprio live-build/live-boot, que já é apt-nativo) para os dois modos, ou um único ISO com a escolha feita em tempo de instalação
- Repositório apt próprio, assinado (chave GPG), hospedando os `.deb` compilados via LFS/hardening

---

## Lista: o que recompilar vs o que vem do Debian `main`

**Observação antes da lista:** já que o gerenciador é apt/dpkg, o caminho mais eficiente não é seguir o LFS "capítulo por capítulo" recompilando cada pacote de userland a partir de tarball — é montar a base com **debootstrap** (puxa os `.deb` já testados do Debian `main` direto) e reservar o esforço "from scratch" pro que é próprio do projeto ou realmente precisa de hardening extra. É basicamente o que Kali, Parrot OS e Trisquel fazem: constroem em cima do debootstrap, não recompilam o Debian inteiro do zero. Isso corta a maior parte do trabalho braçal sem abrir mão do controle nas partes que importam.

### Tier 1 — Recompilar/construir do zero (aqui o LFS entra de verdade)
- **Kernel**: deblob (moldes `linux-libre`) + patches de hardening (`linux-hardened`, `CONFIG_HARDENED_USERCOPY`, `CONFIG_STACKPROTECTOR_STRONG`, `CONFIG_FORTIFY_SOURCE`)
- **hardened_malloc** — não está no Debian main, precisa ser empacotado como `.deb` próprio
- **Painel web de gerenciamento** (`arx-painel`, em Flask) — software desenvolvido pelo projeto, não vem de lugar nenhum
- **App de configurações desktop nativo** — idem, software próprio
- **Perfis AppArmor customizados** por serviço (não é recompilar o AppArmor, é autoria de perfil mesmo)
- **Branding/módulos do Calamares** (tela de seleção Servidor/Cliente)

### Tier 2 — Recompilar com flags extras (opcional, fase avançada — não bloqueia o MVP)
- OpenSSH com defaults mais restritos (menos cifras, sem root login por senha) — **mas começa só configurando via `sshd_config`**, recompilar aqui só se precisar de algo que configuração não resolve
- glibc/openssl — Debian já aplica PIE, RELRO, fortify e stack-protector por padrão via `dpkg-buildflags`; recompilar tem retorno baixo a menos que você queira patches bem específicos (ex: se decidir entrar em grsecurity/PaX RAP mais pra frente)

### Tier 3 — Direto do Debian `main`, sem recompilar
- Userland padrão: coreutils, bash, dbus, systemd (ou outro init, se preferir algo mais enxuto), iproute2 (VLAN 802.1Q já é nativo via `ip link add ... type vlan`), nftables, AppArmor (framework/binários), OpenSSH client/server, Xorg/Wayland + o DE escolhido (XFCE/LXQt)
- Firmware: só o pacote `firmware-free` — nunca `non-free-firmware`

## Repositório apt próprio — arquitetura

### Motor: `aptly`

Em vez de combinar ferramentas soltas (mirror + repo próprio + cache), o `aptly` resolve os três papéis de uma vez e tem **API REST nativa** — o que é o ponto chave, já que o painel web vai controlar tudo:

- **Espelha** repositórios externos (Debian `main` + `main-security`) com agendamento
- **Hospeda** os pacotes próprios (Tier 1: kernel, hardened_malloc, painel, app desktop, perfis AppArmor)
- **Snapshots/publish** — permite versionar e promover pacotes entre canais (staging → stable) sem sobrescrever nada, e dá rollback de graça se uma atualização quebrar algo

### Componentes do repositório

| Componente | Conteúdo | Status real (2026-09-13) |
|---|---|---|
| `main` | Espelho do Debian `main` filtrado (nome + prioridades required/important/standard) + pacotes próprios maduros (`arx-cli`) via repo local `arx-main-local`, mesclados num snapshot só antes de publicar | ✅ **implementado e rodando** |
| `unstable` | Builds novos do Tier 1 (kernel, painel) — hoje é onde `arx-painel` (em desenvolvimento ativo) e o kernel 6.18.45 vivem | ✅ **implementado e rodando** |
| `edge` | O que foi promovido de `unstable` — pensado pro kernel mainline (7.2.x) | ❌ **não criado ainda** — decisão tomada de deixar só planejado por enquanto; exige compilar um kernel novo baseado em 7.2.x (não é só republicar), com hardening/deblob conferidos pra essa série |
| `stable` | O que foi validado o suficiente pra produção — kernel 6.18 LTS é o primeiro candidato | ❌ **não criado ainda** — decisão tomada de promover o kernel (6.18.45, sem mexer no painel, que ainda tem bug ativo) assim que possível |
| `firmware-nonfree` | Microcode Intel/AMD, firmware de GPU não-livre | ❌ não iniciado |

O pipeline sequencial descrito (unstable → edge → stable) é a intenção original — na prática hoje só `unstable` e `main` existem fisicamente no `aptly`; `stable`/`edge`/`firmware-nonfree` são passos futuros claros, não implementados.

### Fluxo servidor → cliente

- O **servidor** é o único ponto que sai pra internet pra sincronizar os mirrors — reduz superfície de exposição de cada máquina cliente (bom pro objetivo de segurança/privacidade)
- **Clientes** (servidor ou desktop) têm `sources.list` apontando só pro IP/hostname do servidor central, nunca direto pros repositórios Debian
- nginx serve o repositório publicado pelo `aptly` (`aptly publish`), com o `.deb` assinado pela chave GPG própria do projeto

### O que fica configurável pelo painel web

- Lista de repositórios externos espelhados e seus componentes
- **Intervalo de sincronização**: predefinições **diário / semanal / quinzenal / mensal / custom** selecionáveis no painel, com **semanal como padrão inicial** (menos tráfego de rede, mais estabilidade). Cada predefinição mapeia direto pra um `OnCalendar` do systemd timer (`custom` aceita a expressão `OnCalendar` diretamente, pra quem quiser algo mais específico)
- Toggle do componente `firmware-nonfree` (ligado/desligado)
- Upload de novas versões dos pacotes próprios (Tier 1) e promoção staging → stable via snapshot
- Histórico de snapshots, com opção de rollback

## Autenticação — AD vs conta local

### Servidor: Samba AD DC

`samba` com o papel `samba-ad-dc` implementa Active Directory (Kerberos + LDAP + DNS) de forma livre, já disponível no Debian `main` (Tier 3 — não precisa recompilar). Precisa entrar no repositório junto: `krb5-kdc` (vem com o provisionamento do Samba) e NTP sincronizado (Kerberos é sensível a relógio, já coberto pelo módulo NTP do painel).

- Módulo novo no painel web: **Domínio** — provisiona o domínio (nome, realm, hostname do DC), gerencia usuários/grupos/OUs do AD (via `samba-tool` por baixo)
- Fica no modo servidor, é opcional ativar — quem não quiser AD, o servidor funciona normalmente sem essa role

### Cliente: escolha na instalação (Calamares)

**Requisito de segurança que vale deixar explícito**: nenhuma ISO/imagem distribuída do Arx OS deve sair com senha de root, hash de senha genérico, ou qualquer credencial fixa pré-definida — usuário/senha (ou credencial de AD) sempre são definidos pelo próprio instalador, durante a instalação, igual ao Debian padrão faz. É um erro clássico de imagem "pronta" (Raspberry Pi OS já foi criticado anos por isso) fácil de esquecer justamente na hora de automatizar o build da ISO.

Nova tela no instalador, equivalente à tela de domínio do Windows, com duas opções mutuamente exclusivas:

- **"Usar servidor de domínio"** → pede endereço do domínio + credenciais, executa `realm join` (via `sssd` + `realmd`, que já falam AD nativamente) como último passo da instalação
- **"Criar conta local"** → fluxo padrão do módulo `users` do Calamares, sem AD

Login via AD nos dois modos (servidor e desktop) passa a usar `sssd` como backend de autenticação (PAM + NSS), que é o caminho padrão pra Linux falar com AD sem precisar de `winbind` legado.

## Painel web — arquitetura de privilégios (importante pro AppArmor)

O painel precisa mexer em rede/VLAN, firewall, apt, e Samba AD — ou seja, precisa de poder de root pra boa parte de suas funções. Dar isso tudo direto pro processo Flask/gunicorn que fica exposto na web é ruim pra confinamento (um perfil AppArmor "solto" o suficiente pra cobrir tudo isso não protege muita coisa). O padrão usado por painéis desse tipo (Cockpit, por exemplo) é separar em duas peças:

- **Web UI (Flask/gunicorn)**: roda sem privilégio nenhum, só fala com o helper via socket local/D-Bus. Perfil AppArmor bem apertado.
- **Helper privilegiado**: processo pequeno, rodando como root, com uma lista fechada de comandos que ele tem permissão de executar (`nft`, `ip`, `samba-tool`, `aptly`, `systemctl restart <serviços específicos>`) — nada além disso. Perfil AppArmor também apertado, só que liberando esses binários específicos.

Isso é o que torna os dois perfis abaixo possíveis de escrever com confiança, em vez de um perfil enorme e permissivo.

---

## O que usamos como base — resumo

- **Userland**: Debian `main` via `debootstrap` (não LFS capítulo-por-capítulo — ver Fase 0.1)
- **Gerenciador de pacotes**: apt/dpkg
- **Kernel**: deblobbed (linux-libre) + hardening (`linux-hardened`) — versão a decidir entre 7.2 e 6.18, ver Fase 1
- **MAC/sandboxing**: AppArmor — **perfis do `arx-painel` (webui + helper) prontos e em modo enforce**; nginx, Samba AD e sshd ainda não têm perfil próprio
- **Rede/firewall**: nftables + VLAN 802.1Q nativo
- **Repositório**: `aptly` (mirror + repo próprio + snapshots), publicado via nginx, restrito à rede interna
- **Autenticação**: Samba AD DC (opcional) + `sssd`, ou conta local — escolha na instalação
- **Painel servidor**: Flask, arquitetura privilégio-separada (webui + helper)
- **Desktop**: KDE Plasma, com Baloo/KUserFeedback desligados por padrão
- **Instalador**: Live ISO + Calamares customizado

## Ordem de implementação prática

1. ✅ Decidir a versão do kernel — `stable`=6.18 LTS, `edge`=7.2 mainline (planejado, não construído)
2. ✅ Deblob + hardening do kernel — 6.18.45 compilado e testado pros dois perfis (server/desktop)
3. ✅ Montar a base com `debootstrap` — rootfs mestre pronto, com rede/DNS de fábrica
4. ✅ Empacotar o resto do Tier 1 — `arx-painel` **completo na Fase 2, e 5/6 da Fase 5 de robustez** (ver roadmap próprio), `arx-cli` publicado; `hardened_malloc` e app desktop **não iniciados**; perfis AppArmor do **próprio painel** escritos e em modo enforce — perfis de outros serviços (nginx, Samba AD, sshd) ainda não iniciados
5. ✅ Repositório apt próprio rodando — `main`+`unstable` publicados via nginx; `stable`/`edge`/`firmware-nonfree` ainda não criados (ver seção de componentes)
6. ✅ Samba AD DC — provisionamento funcionando via painel; sshd hardened não conferido especificamente ainda
7. ⚠️ Perfis AppArmor — **os dois do `arx-painel` (webui + helper) prontos e em enforce**; os de nginx/Samba AD/sshd (decisão original da Fase 0) ainda não escritos
8. ❌ KDE Plasma sobre a base desktop — **não iniciado**, é o próximo bloco grande de trabalho (Fase 3)
9. ⚠️ Branding/identidade — nome decidido (Arx OS), aplicado no que já existe; Plymouth/SDDM/Calamares dependem da Fase 3
10. ❌ Instalador Calamares — não iniciado, depende da Fase 3
11. ❌ Gerar ISOs e testar em VM — não iniciado

## CLI `arx` e loja de aplicativos (decidido em 2026-09-12, não previsto originalmente)

Componente separado do painel (`arx-cli`, pacote próprio publicado no `main`):

- **`arx`** (`/usr/local/bin/arx`): wrapper fino sobre o `apt` — cobre `update/upgrade/dist-upgrade/full-upgrade/install/remove/purge/autoremove/search/list/show`, repassa argumentos integralmente (`"$@"`), identifica a ação pelo primeiro argumento sem `-`, termina com `exec apt "$@"`. Autocomplete reaproveitando a mesma função do `apt` (`_apt` via `bash-completion`), sem manter lista própria de pacotes
- **Decisão de arquitetura da loja de aplicativos**: o repositório Arx atua como filtro de instalação — pacote do repositório próprio instala **sem senha** de administrador; pacote de fora (repo externo, `.deb` avulso) **exige** senha do administrador do domínio
- **Mecanismo planejado** (ainda não implementado, depende da Fase 3): PolicyKit (`polkit`) com regra customizada em JS, autorizando `org.freedesktop.packagekit.package-install` sem senha só quando o pacote vem do source Arx pinado e o usuário é de domínio — frontend gráfico natural seria o Discover do KDE (via PackageKit → polkit)
- **Dependência crítica**: polkit só valida contas locais por padrão — só funciona de verdade com o desktop integrado ao domínio via `sssd`/PAM, que ainda não existe no projeto
- Rascunho da regra (`10-arx-repo-installs.rules`) já escrito, mas marcado explicitamente como incompleto — falta a checagem real de origem do pacote, e falta a integração desktop+AD

## Grupos de máquinas — software por grupo (ex: Lab 1 / Lab 2)

Viável pra agora — reaproveita 100% da infra já desenhada (apt + `aptly` + arquitetura servidor→cliente), não é feature que precisa esperar uma versão futura:

- Painel define **grupos** (nome livre — "Lab 1", "Financeiro", "Recepção" — associados a máquinas por hostname, faixa de IP/VLAN, ou registro manual no join)
- Cada grupo tem uma lista de pacotes desejada; o painel gera um **metapacote** `.deb` por grupo (só um `Depends:` com a lista, sem conteúdo próprio) e publica no componente `stable`/`edge` conforme o canal do grupo
- Cliente já está apontado pro servidor (arquitetura decidida na Fase 2) — um agente simples (`systemd timer`, reaproveitando o mesmo mecanismo do sync de atualização) confere qual metapacote o grupo da máquina usa e roda `apt install`/`apt upgrade` normalmente
- Trocar o grupo de uma máquina, ou mudar o que está no grupo, é só editar a lista no painel e esperar o próximo check-in — sem tocar em nada na máquina cliente

Não precisa de tecnologia nova nenhuma — é a mesma tubulação do módulo de Atualizações, só que parametrizada por grupo em vez de igual pra todo mundo.



Pra deixar a "cara de sistema empresarial/acadêmico" (item 9):

- **Nome da distro: decidido — "Arx OS"** (torre/fortaleza formando um "A", tagline "Secure · Private · Free"). Já em uso em: `/etc/os-release` do rootfs mestre, logo do painel web (horizontal + ícone), codinomes de versão (`0.1 "Masada"`, fortalezas históricas), distribuição do aptly (`masada`)
- **Ainda pendente** (bloqueado por Fase 3/Desktop, que ainda não começou): tema de boot do Plymouth, tema do SDDM/Plasma (global theme), `branding.desc` do Calamares — todos dependem do ambiente gráfico existir primeiro
