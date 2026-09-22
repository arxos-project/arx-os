# Arx OS

**Secure Private Free**

Distribuição Linux baseada em Debian (não é LFS puro — usa `dpkg`/`apt`), com foco em segurança e privacidade, voltada para uso acadêmico, empresarial e doméstico.

Versão atual: **0.1 "Masada"**

## Sobre o nome

*Arx* — do latim, "fortaleza" — representada no logo por uma torre formando a letra "A".

> Existe outro projeto de mesmo nome (`thearxos`, uma distro Arch/BlackArch focada em pentest). O Arx OS aqui documentado é um projeto independente, com proposta e base técnica completamente diferentes.

## Perfis

- **Servidor** — `systemd-networkd`, painel web ([`arx-painel`](https://github.com/arxos-project/arx-painel)), LUKS por padrão, pode atuar como Samba AD DC
- **Desktop/Cliente** — KDE Plasma, `NetworkManager`, sem LUKS forçado (voltado também para PCs fracos que não rodam Windows 11 confortavelmente — perfil "Terminal Público"/quiosque)

## Canais de pacotes

| Canal | Descrição |
|---|---|
| `stable` | Kernel LTS, testado |
| `edge` | Mais recente, sem LTS |
| `unstable` | Desenvolvimento ativo |

Fluxo de promoção fixo: `unstable → edge → stable`, nunca pulando etapa.

## Componentes do projeto

- [`arx-painel`](https://github.com/arxos-project/arx-painel) — painel web de administração
- [`arx-cli`](https://github.com/arxos-project/arx-cli) — wrapper de linha de comando

## Repositório de pacotes

`deb http://arxos.is-a.dev/repo stable main`

Chave pública GPG distribuída via pacote `arx-archive-keyring` (ver instruções nos READMEs de `arx-painel`/`arx-cli`).

## Política de firmware

`contrib`/`non-free`/`non-free-firmware` são sempre **opt-in** — o kernel permanece *deblobbed* (sem firmware não-livre) por padrão, sem exceção. A escolha é feita no instalador (Calamares, modo Cliente) ou via toggle na interface gráfica pós-instalação.

## Status

Projeto em desenvolvimento ativo. Infraestrutura de build e repositório público em preparação.

## Licença

[GNU GPLv3](LICENSE)

## Contato

arxos.project@gmail.com
