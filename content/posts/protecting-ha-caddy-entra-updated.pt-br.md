---
title: "Protegendo o Home Assistant com Caddy e Microsoft Entra ID"
date: 2025-09-29T20:45:00-04:00
draft: false
toc: true
tags: ["home-assistant", "caddy", "entra-id", "reverse-proxy", "diy"]
---

Na minha eterna missão de organizar o meu home lab, finalmente resolvi algo que me incomodava há um tempo: **como expor o Home Assistant para fora da minha rede de forma segura, sem logins duvidosos ou avisos de certificados autoassinados.**  

Primeiro de tudo: eu tinha algumas opções para o proxy reverso — os suspeitos de sempre como nginx ou Traefik — mas escolhi o **Caddy**. Por quê? Sinceramente, porque é muito mais fácil. A sintaxe de configuração é limpa, ele faz a coisa certa na maior parte do tempo sem precisar brigar, e a comunidade/documentação são excelentes. Com nginx eu sempre sinto que estou caçando mais uma linha de `proxy_set_header`. Com o Caddy, simplesmente... funciona.  

E o proxy reverso em si? Não é só para ter URLs bonitinhas. Ele me dá uma barreira de segurança dura. Posso forçar **login da Microsoft + MFA** antes de qualquer um acessar meus dashboards, e ainda manter uma segmentação forte entre a **VLAN principal** e a **VLAN de IoT**, que é muito mais vulnerável. Mesmo que algum dispositivo do lado IoT fosse comprometido, o proxy reverso garante que o acesso aos serviços críticos continue protegido e registrado.  

O projeto do fim de semana, então, foi colocar o Caddy na frente do Home Assistant e integrá-lo ao **Microsoft Entra ID** para autenticação. Agora, antes mesmo de ver o meu dashboard, eu passo pelo login da Microsoft.  

A melhor parte? Tudo rodando no meu servidorzinho Docker, usando a versão gratuita do Entra e os certificados da DigiCert que eu já tinha. Sem assinaturas extras, sem depender de nuvem — só um proxy reverso sólido com SSO que parece coisa de ambiente corporativo.  

Próximo passo na lista? Enviar todos esses logs para o **Microsoft Sentinel** para começar a criar detecções e alertas. Mas isso fica para outro fim de semana. 😉

---

## Tá, mas por que fazer isso?

- **Proxy reverso centralizado**: URLs amigáveis como `https://ha.domain.com`.
- **TLS com DigiCert** (nada de avisos chatos do navegador!).
- **Autenticação com Entra**: login Microsoft + MFA antes de qualquer acesso.
- Funciona tanto **dentro quanto fora da rede** (com port-forwarding no UDM Pro ou via VPN).

---

## Arquitetura

```
Navegador → Caddy (TLS) → oauth2-proxy (OIDC auth) → Home Assistant
                               ↑
                         Login no Entra ID
```


## O que é um Reverse Proxy?

Um **reverse proxy** é um servidor que fica entre o usuário e os serviços internos.  
Em vez de o usuário acessar diretamente o **Home Assistant** (ou outro app) no IP e porta, ele fala com o reverse proxy (Caddy, no meu caso).  

O reverse proxy:
- Recebe a conexão do usuário (normalmente HTTPS na porta 443).
- Verifica autenticação, certificados, regras de segurança, etc.
- Encaminha a requisição para o serviço interno correto.
- Retorna a resposta para o usuário como se viesse do próprio domínio.

Isso simplifica muito (um único ponto de entrada) e aumenta a segurança, porque você pode colocar autenticação forte e TLS centralizados.

### Diagrama

![Diagrama do Reverse Proxy](/images/reverse-proxy-diagram.png)


No meu caso, o reverse proxy também faz **integração com Entra ID** (login Microsoft + MFA) e garante segmentação entre a rede principal e a IoT VLAN.


---

## Pré-requisitos

- Uma máquina Linux rodando Docker (a minha é o `Srv-Core-Dockerland` em `10.0.0.5`).
- Home Assistant acessível em `10.0.1.2:8123`.
- Certificados DigiCert para `ha.domain.com` (wildcard também funciona).
- Um domínio (ex: `domain.com`) com DNS interno (Pi-hole/Unbound) e DNS externo apontando para o seu UDM Pro.

---

## Passo 1 — Registro de App no Entra ID

1. Vá até o [Azure Portal](https://portal.azure.com) → **Entra ID** → *App registrations* → *New registration*.
2. Nome: `HomeLab Reverse Proxy`.
3. Contas: *apenas este diretório organizacional*.
4. Redirect URI:  
   ```
   https://ha.domain.com/oauth2/callback
   ```
5. Salve → copie o **Application (client) ID** e o **Directory (tenant) ID**.
6. Em *Certificates & secrets* → crie um **client secret** (guarde o valor).
7. Em *Authentication* → habilite **ID tokens** + **Access tokens**.

---

## Passo 2 — Docker Compose

`/srv/docker/docker-compose.yml`:

```yaml
version: "3.8"

services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
      - ./certs:/certs:ro
    networks: [rpnet]

  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
    container_name: oauth2-proxy
    restart: unless-stopped
    env_file: ./oauth2-proxy/env
    networks: [rpnet]

networks:
  rpnet:
    driver: bridge
```

---

## Passo 3 — Configuração do oauth2-proxy

`/srv/docker/oauth2-proxy/env`:

```env
OAUTH2_PROXY_PROVIDER=oidc
OAUTH2_PROXY_OIDC_ISSUER_URL=https://login.microsoftonline.com/<TENANT_ID>/v2.0
OAUTH2_PROXY_CLIENT_ID=<APP_CLIENT_ID>
OAUTH2_PROXY_CLIENT_SECRET=<APP_CLIENT_SECRET>
OAUTH2_PROXY_COOKIE_SECRET=<BASE64_32>   # openssl rand -base64 32
OAUTH2_PROXY_REDIRECT_URL=https://ha.domain.com/oauth2/callback
OAUTH2_PROXY_UPSTREAMS=http://10.0.1.2:8123
OAUTH2_PROXY_COOKIE_SECURE=true
OAUTH2_PROXY_COOKIE_SAMESITE=lax
OAUTH2_PROXY_COOKIE_DOMAIN=ha.domain.com
OAUTH2_PROXY_WHITELIST_DOMAINS=.domain.com
OAUTH2_PROXY_SET_XAUTHREQUEST=true
OAUTH2_PROXY_PASS_ACCESS_TOKEN=true
OAUTH2_PROXY_SCOPE=openid profile email
OAUTH2_PROXY_SKIP_PROVIDER_BUTTON=true
```

⚠️ **Pegadinha**: o cookie secret precisa ter exatamente 16, 24 ou 32 bytes. Não coloque aspas ou comentários na mesma linha do arquivo `.env`. Fiz isso e perdi uma hora caçando erro 🥺

---

## Passo 4 — Configuração do Caddy

`/srv/docker/caddy/Caddyfile`:

```caddyfile
ha.domain.com {
  tls /certs/ha.crt /certs/ha.key

  reverse_proxy oauth2-proxy:4180 {
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    header_up X-Forwarded-Proto {scheme}
  }
}
```

---

## Passo 5 — Configuração no Home Assistant

Agora você precisa dizer ao HA para confiar no proxy. Edite o `configuration.yaml` e adicione:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.0.0.5          # Srv-Core-Dockerland
    - 172.16.0.0/12     # rede bridge do Docker
```

---

## Passo 6 — Acesso externo

- No UDM Pro: faça o forward da porta **TCP/443** → `10.0.0.5`.
- Configure seu provedor de DNS externo: crie um A record (`ha.domain.com`) → IP público (WAN).
- Configure seu DNS interno (ex: Pi-hole): (`ha.domain.com`) → `10.0.0.5`.

---

## Problemas comuns e soluções

- **CSRF token mismatch**  
  Certifique-se que o Caddy encaminha os headers `Host` e `X-Forwarded-Proto`, e defina `OAUTH2_PROXY_COOKIE_DOMAIN` para o seu hostname.
- **Unauthorized em vez de redirecionar**  
  Isso acontece com `forward_auth`. Use o padrão mais simples *auth-proxy* mostrado aqui.
- **403 após login**  
  Verifique se o Redirect URI no Entra bate exatamente com o valor em `OAUTH2_PROXY_REDIRECT_URL`.

---

## Conclusão

Com essa configuração eu consigo acessar o **Home Assistant** de qualquer lugar com segurança, usando **login Microsoft + MFA**. Nada de portas expostas direto, nada de telas de login frágeis, e TLS da DigiCert ponta a ponta.  

Esse padrão também funciona para outros serviços (HomeBridge, media servers, etc.) — basta adicionar novos subdomínios, certificados e blocos no Caddy. Simples e seguro!

---
