# edge-proxy

Proxy de borda compartilhado da VPS: **único dono das portas 80/443**, Caddy com TLS automático.
Substitui o `radar_proxy` (nginx do repo radar-backend) e o certbot standalone, cuja renovação
quebrava porque a porta 80 ficava ocupada (o cert do radar staging venceu em 23/09/2026 por isso).

| Arquivo | Papel |
|---|---|
| `compose.yaml` | contêiner `edge_caddy`, redes `edge` (criada aqui) e `radar_gateway` (externa), volume de certificados |
| `conf/Caddyfile` | opções globais + `import sites/*.caddy` |
| `conf/sites/radar-staging.caddy` | `api.staging.painelpolitico.com.br` → `staging_nginx:80` |
| `conf/sites/gamedle.caddy.off` | Gamedle (renomear para `.caddy` quando o DNS existir) |
| `conf/sites/ip-gamedle.caddy` | acesso provisório `http://187.77.254.161/gamedle` (noindex) |
| `.github/workflows/deploy.yml` | valida no CI, sincroniza para `/srv/edge-proxy`, recarrega o Caddy |

## Fluxo normal (sempre pelo repo — não editar na VPS)
1. Editar/adicionar `conf/sites/<projeto>.caddy` e dar push na `main`.
2. O Actions valida a config, faz rsync para `/srv/edge-proxy` e roda `docker compose up -d` + `caddy reload` (sem queda).

## Adicionar um projeto
1. No projeto: ligar o serviço web à rede externa `edge` com um alias (ex.: `aliases: [meuapp]`).
2. Aqui: `conf/sites/meuapp.caddy` com `dominio.com { reverse_proxy meuapp:PORTA }` → push.

## Migração inicial a partir do radar_proxy (uma vez)
Actions → **Deploy edge-proxy** → *Run workflow* (a caixa `cutover` já vem marcada). **Não use Re-run** de um run de push: ele repete os parâmetros do push (sem cutover). O workflow:
1. valida a config e baixa a imagem (sem afetar nada);
2. derruba a stack `radar-proxy` (libera 80/443) e sobe o Caddy;
3. espera até 120 s por HTTPS **válido** (sem `-k`) em `CHECK_URLS`;
4. se falhar, **rollback automático**: derruba o Caddy e sobe o `radar_proxy` de novo.

Enquanto o `radar_proxy` ocupar as portas, pushes normais só sincronizam os arquivos (aviso no log).

## Secrets (mesmos dos outros repos da VPS)
`VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `VPS_PORT` (opcional) + environment `production`.

⚠️ Nunca `docker compose down -v`: apaga os certificados (reemitir em massa pode bater rate limit do Let's Encrypt).
Depois da migração, o `certbot.timer` do host pode ser desativado (`systemctl disable --now certbot.timer`).
