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
| `conf/sites/domecap.caddy` | `http://187.77.254.161:8080` → `domecap_nginx:80` (rede `domecap_domecap`) |
| `.github/workflows/deploy.yml` | valida no CI, sincroniza para `/srv/edge-proxy`, recarrega o Caddy |

## Fluxo normal (sempre pelo repo — não editar na VPS)
1. Editar/adicionar `conf/sites/<projeto>.caddy` e dar push na `main`.
2. O Actions valida a config, faz rsync para `/srv/edge-proxy` e roda `docker compose up -d` + `caddy reload` (sem queda).

## Projetos por porta (sem domínio)
Cada porta pública é publicada **aqui** (`compose.yaml` → `ports`) e roteada por um `conf/sites/<projeto>.caddy`
com `http://:PORTA { reverse_proxy <conteiner>:<porta-interna> }`. O projeto para de publicar a porta no
próprio compose (fica só em `127.0.0.1`, para debug). O Caddy entra na rede Docker do projeto (`networks`).

**Trava de porta:** se uma porta deste compose ainda estiver em uso por outro contêiner, o deploy **não recria**
o `edge_caddy` (ele não subiria e derrubaria todos os sites) — só recarrega a config e avisa. Migração de um
projeto por porta, portanto:
1. push aqui (rota + porta + rede) → deploy avisa "porta ocupada", nada cai;
2. deploy do projeto liberando a porta (fica fora do ar até o passo 3);
3. rodar este workflow de novo (ou `docker compose up -d` em `/srv/edge-proxy`) → o Caddy publica a porta.

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

**Migração feita em 2026-10-20 02:29 UTC** (executada com os comandos deste workflow, direto na VPS): o
radar staging voltou a ter HTTPS válido (o cert antigo tinha vencido em 23/09) e o Gamedle ficou público em
`http://187.77.254.161/gamedle/`. Os dois runs manuais anteriores não migraram por causa de um bug: o
`docker compose run` lia o stdin, que era o próprio script (`bash -s <<EOF`), e "engolia" o resto. Corrigido
com `run -T ... </dev/null`.

## Diagnóstico
Cada deploy grava a saída do passo "Aplicar" em `/srv/edge-proxy/deploy.log` (protegido do rsync).

## Secrets (mesmos dos outros repos da VPS)
`VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `VPS_PORT` (opcional) + environment `production`.

⚠️ Nunca `docker compose down -v`: apaga os certificados (reemitir em massa pode bater rate limit do Let's Encrypt).
Depois da migração, o `certbot.timer` do host pode ser desativado (`systemctl disable --now certbot.timer`).
