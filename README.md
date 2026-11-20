# Dúvidas Técnicas

Sistema de perguntas e respostas para exercícios e problemas técnicos. Estudantes entram com o Discord, abrem tópicos descrevendo a dúvida e respondem tópicos dos colegas. **Somente o autor do tópico** pode marcar uma resposta como **Solução Aceita**, que sobe para o topo e muda o selo do tópico para **Resolvido**.

> **Chave de auditoria: `P26-2C-DC19-34DD-D6E4-3F3F`**

| | |
|---|---|
| Autor | Guilherme Borges Szimzek |
| RA | 2026109288 |
| Projeto online | ATUALIZE: `https://duvidas-tecnicas.pages.dev` (URL final do Cloudflare Pages) |
| Repositório | ATUALIZE: URL deste repositório no GitHub |
| Licença | [MIT](LICENSE) |

## Stack

- **Cliente:** HTML5, CSS3 e JavaScript puro (Vanilla JS), sem frameworks de interface e sem CDN.
- **Servidor:** Cloudflare Pages Functions (roteamento por arquivo em `functions/`).
- **Banco:** Cloudflare D1 (SQLite relacional): `users`, `sessions`, `topics`, `answers`.
- **Login:** OAuth2 do Discord (Authorization Code, scope `identify`), importando `user_id`, nome de usuário e avatar.

## Regra de negócio central

A permissão de marcar "Solução Aceita" é validada na Cloudflare Function (`functions/api/topics/[id]/accept.js`), nunca no navegador:

1. A sessão (cookie `sid`) identifica o usuário; o `user_id` do Discord vem do D1.
2. A Function lê o tópico e compara `session.user_id` com `topics.author_id`. Diferente: **403**, nada muda.
3. Confirma que a resposta pertence ao tópico (senão **422**).
4. Grava com um único `UPDATE ... WHERE id = ? AND author_id = ? AND EXISTS (...)`, atômico, e confere `meta.changes`.
5. O tópico passa a `status = 'resolved'`, guarda `accepted_answer_id` e `resolved_at`. A listagem de respostas devolve a aceita primeiro.

Outras regras no servidor: o autor não responde o próprio tópico (403), tópico resolvido não recebe respostas (409), limites de tamanho (422), `Origin` e `Content-Type` verificados nos POST (CSRF), consultas sempre parametrizadas.

## Funcionalidades

- Lista de tópicos com filtros (Todos, Abertos, Resolvidos), busca por título e paginação.
- Novo tópico e novas respostas com suporte a blocos de código (três crases) e código inline, renderizados via DOM (sem `innerHTML`, sem XSS).
- Selo Aberto/Resolvido, destaque visual da Solução Aceita, botão "Marcar como solução" só para o autor.
- Layout responsivo, tema claro/escuro, foco visível, textos em pt-BR.

## Estrutura

```
public/                 index.html, style.css, app.js
functions/_lib/         http, session, discord, validate
functions/api/          auth/{login,callback,logout}, me, topics/...
migrations/0001_init.sql
tests/                  api.test.mjs + helpers/d1-shim.mjs
docs/CONTRATO-API.md    contrato da API
.claude/agents/         3 agentes (papéis)
.claude/skills/         4 skills usadas pelos agentes
AGENTES.md              agentes, harness e fluxo de trabalho
TUTORIAL.md             passo a passo pelos sites: Discord, Cloudflare, GitHub e AVA
```

## Publicação

Sem instalar nada: o deploy é feito pelos sites do GitHub e da Cloudflare (Pages conectado ao repositório, banco D1 criado no painel). O passo a passo está em [TUTORIAL.md](TUTORIAL.md).

Configuração no painel da Cloudflare: binding D1 `DB`, variável `DISCORD_CLIENT_ID` e secret `DISCORD_CLIENT_SECRET`.

## Testes

```bash
node --test tests/api.test.mjs   # 46 testes (opcional, exige Node 22+; sem dependências)
```

Os testes importam as Functions diretamente e usam `node:sqlite` em memória no lugar do D1, com o `fetch` do Discord simulado. Cobrem autenticação, permissões, validação, injeção de SQL, o fluxo de Solução Aceita e uma varredura estática do front-end.

## Agentes

Três agentes especializados (backend, front-end e QA/validação) construíram e validaram o projeto. Detalhes, harness, skills e relatório de testes em [AGENTES.md](AGENTES.md).

## Licença

MIT. Veja [LICENSE](LICENSE).
