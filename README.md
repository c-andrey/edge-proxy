# edge-proxy

Proxy de borda compartilhado da VPS: **único dono das portas 80/443**, Caddy com TLS automático.
Substitui o `radar_proxy` (nginx do repo radar-backend) e o certbot standalone.

| Arquivo | Papel |
|---|---|
| `compose.yaml` | contêiner `edge_caddy`, redes `edge` (criada aqui) e `radar_gateway` (externa), volume de certificados |
| `Caddyfile` | opções globais + `import sites/*.caddy` |
| `sites/radar-staging.caddy` | `api.staging.painelpolitico.com.br` → `staging_nginx:80` |
| `sites/gamedle.caddy.off` | Gamedle (ativar quando o DNS existir) |

## Adicionar um projeto
1. No projeto: ligar o serviço web à rede externa `edge` com um alias (ex.: `aliases: [meuapp]`).
2. Aqui: criar `sites/meuapp.caddy` com `dominio.com { reverse_proxy meuapp:PORTA }`.
3. `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile && docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`

## Migração a partir do radar_proxy (feita uma vez)
```bash
cd /srv/edge-proxy
docker compose pull && docker compose run --rm --no-deps --entrypoint caddy caddy validate --config /etc/caddy/Caddyfile
docker compose -p radar-proxy -f /srv/radar-backend-staging/docker-compose.proxy.yml down   # libera 80/443
docker compose up -d && docker compose logs -f caddy      # aguardar "certificate obtained successfully"
curl -sI https://api.staging.painelpolitico.com.br | head -1
```
**Rollback** (segundos): `docker compose down && docker compose -p radar-proxy -f /srv/radar-backend-staging/docker-compose.proxy.yml up -d`

⚠️ Não use `docker compose down -v` (apaga os certificados; reemitir em massa pode bater rate limit do Let's Encrypt).
