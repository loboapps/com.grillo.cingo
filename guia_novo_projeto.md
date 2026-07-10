# Guia — Iniciando um Novo Projeto no Padrão APT

> Este documento consolida os padrões usados no projeto APT para servir de ponto de partida para qualquer novo projeto React + Supabase seguindo a mesma estrutura.

---

## 0. Perguntas obrigatórias antes de começar

Antes de escrever qualquer código, **sempre perguntar ao usuário**:

1. **Qual o prefixo do projeto?** — usado em todas as tabelas, RPCs, roles e classes de tema (ex: `apt_` para o projeto APT). Definir também o sufixo de admin (ex: `apt_admin_`).
2. **O projeto é público ou privado?**
   - **Público**: dados de leitura acessíveis por qualquer pessoa (`anon` + `authenticated`) — ex: ranking de um torneio que outras pessoas acompanham. RLS libera `SELECT` para `public`.
   - **Privado**: só o dono/usuários autenticados específicos podem ler qualquer coisa — RLS restringe `SELECT` a `authenticated` (e possivelmente a um filtro de "dono do registro"), sem exposição a `anon`.

Perguntas derivadas que também vale confirmar cedo (podem ter resposta óbvia, mas evitam retrabalho):

3. As entidades principais são identificadas por **nome/string** (ex: torneio "APT XXIV") ou fazem mais sentido com **UUID**? Isso muda toda a modelagem de PK/FK e as assinaturas das RPCs.
4. Existe um conceito de **admin** separado de usuário comum, ou todo usuário autenticado tem os mesmos poderes?
5. O projeto precisa de **PWA** (instalável, service worker) ou é só uma web app comum?

---

## 1. Stack técnica padrão

- **Frontend**: React 18 + TypeScript + Vite + TailwindCSS
- **Roteamento**: React Router v6, configurado em `src/main.tsx`
- **Backend**: Supabase (Postgres + Auth + RPCs), client via `@supabase/supabase-js`
- **Ícones**: `lucide-react`
- **Deploy**: Vercel, com rewrite de SPA (`vercel.json`)
- **PWA (opcional)**: service worker registrado em `main.tsx`, componente `InstallPrompt`

`vite.config.ts` de referência:
- Alias `@` → `src/`
- `manualChunks` separando `vendor` (react/react-dom) e o SDK do Supabase
- `server.host = true` para testar em dispositivos na rede local

`tailwind.config.js`: cores customizadas sob um namespace curto do projeto (ex: `apt: { 100..900 }`, do mais claro ao mais escuro), fontes de display/sans dedicadas.

### Padrões de código

- **TypeScript estrito, `any` nunca é permitido** — nem em componentes, camadas, callbacks, casts ou em qualquer lugar. Usar tipos exportados pela biblioteca (ex: `BarCustomLayerProps<D>` do `@nivo/bar`), generics, ou `unknown` + narrowing quando não houver tipo preciso. Para incompatibilidades de shape inevitáveis, cast através de um tipo concreto (`x as unknown as (v: number) => number`), nunca `as any`.
- **Todos os dados carregados via JSONB de RPCs do Supabase** — nunca fazer REST direto em tabela (`supabase.from(...).select(...)`) a partir do frontend. Toda leitura/escrita passa por função (`rpc(...)`).
- **Ambiente único: produção.** Sem staging/homologação — o projeto roda direto contra o banco de produção. Isso implica cuidado redobrado em qualquer mudança de schema ou dado (ver seção 11).

---

## 2. Estrutura de pastas

```
src/
  components/     # Nav, SubNav, Toast, PrivateRoute, componentes de UI reutilizáveis
  contexts/       # AuthContext
  pages/          # páginas públicas
  pages/admin/    # páginas administrativas (forms de criação/edição)
  services/       # <prefixo>Service.ts — única porta de entrada para o Supabase
  types/          # tipos TS espelhando os retornos das RPCs
  utils/          # helpers (ex: mapa de ícones)
.supabase/
  tables/         # 1 arquivo .sql por tabela — fonte de verdade local do schema
  functions/      # 1 arquivo .sql por função/RPC — fonte de verdade local
docs/             # documentação de referência (fluxos, modelo de segurança, este guia)
```

`.supabase/tables` e `.supabase/functions` são a **fonte de verdade local** do schema — todo o banco remoto deve ser reproduzível a partir desses arquivos.

---

## 3. Convenções de nomenclatura

- Prefixo único em **tudo**: tabelas, views, RPCs, roles de negócio, classes de tema Tailwind.
- RPCs de escrita/admin usam um sufixo adicional (ex: `apt_admin_...`, `apt_atualizar_...`, `apt_iniciar_...`).
- RPCs de leitura usam um prefixo consistente (ex: `apt_load_...`).
- Entidades de negócio centrais podem ser identificadas por **nome (string)** em vez de UUID quando isso reflete melhor o domínio (ex: nomes de torneio, etapa, jogador são únicos e legíveis) — mas registros derivados/eventos (ex: resultados) ainda usam `id` UUID.
- Se identificação por nome for adotada, **toda tabela derivada deve ter a entidade-raiz na chave** (ex: `(torneio, etapa)` como parte da PK) para permitir filtro seguro — ver seção 7.

---

## 4. Modelo de segurança Supabase (RLS + Funções)

Padrão de referência (ajustar SELECT conforme público/privado, definido na seção 0):

- **Todas as tabelas** do prefixo têm RLS habilitado.
  - **Público**: `SELECT` liberado para `public` (anon + authenticated); nenhuma política de `INSERT/UPDATE/DELETE` — toda escrita passa só por RPC.
  - **Privado**: `SELECT` restrito a `authenticated`, tipicamente com filtro adicional (`id = auth.uid()` ou equivalente); mesma regra de escrita só via RPC.
- **Funções de leitura** (`<prefixo>_load_*`): `SECURITY INVOKER` — respeitam o RLS da tabela subjacente. Acessíveis por quem o RLS permitir.
- **Funções de escrita/admin** (`<prefixo>_admin_*`, `<prefixo>_atualizar_*`, `<prefixo>_iniciar_*` etc.): `SECURITY DEFINER`; `EXECUTE` concedido só a `authenticated`; toda função chama a função de checagem de admin (`<prefixo>_is_admin()`) na primeira linha.
- **Trigger functions** (ex: atualização automática de totais, criação de perfil ao registrar usuário, validações): sem `EXECUTE` para nenhum role público — chamadas apenas pelo mecanismo de trigger.
- **Views**: `SECURITY INVOKER` — herdam o RLS das tabelas de origem. Se alguma view precisar agregar dados sensíveis, considerar `SECURITY DEFINER` com cautela.

### Tabela de perfis + trigger de novo usuário

- Tabela `<prefixo>_profiles (id uuid PK → auth.users, email, role default 'regular', created_at)`.
- Trigger `AFTER INSERT ON auth.users` chama `<prefixo>_handle_new_user()` (`SECURITY DEFINER`, sem `EXECUTE` para ninguém) que faz `INSERT ... ON CONFLICT DO NOTHING` em `<prefixo>_profiles`.
- `<prefixo>_is_admin()`: `SECURITY DEFINER`, `STABLE`, consulta `<prefixo>_profiles` por `role = 'admin'`. Pode ser exposta a `public` (retorna `false` para não-admin — seguro expor), mas normalmente revogada de `anon` já que só faz sentido para fluxos autenticados.

### Fluxo de autorização para escrita

```
chamada via /rest/v1/rpc/<prefixo>_admin_algo
  → anon?                  → 403 (sem EXECUTE grant)
  → authenticated, não admin? → { success: false, error: 'Unauthorized' } (via is_admin())
  → authenticated, admin?  → executa com SECURITY DEFINER
```

### Variante: projeto single-user (sem multi-tenant/admin)

Para projetos pessoais com um único usuário (sem conceito de "outros usuários" ou papéis admin/regular), o padrão acima pode ser simplificado:

- Toda função resolve o usuário como `v_usuario_id := coalesce(auth.uid(), <prefixo>_fnc_load_defaultuser())` — usa `auth.uid()` quando autenticado, cai para um usuário padrão fixo (hardcoded) em contexto não autenticado/de serviço (ex: cron).
- **Nunca receber `user_id` como parâmetro vindo do frontend** — sempre resolver no servidor.
- A maioria das funções usa `SECURITY INVOKER` (respeita RLS) + `SET search_path = 'public'` (evita schema injection); só triggers usam `SECURITY DEFINER`, por precisarem rodar em contexto de sistema.
- RLS de cada tabela: policy única `user_id = auth.uid()` (com fallback ao usuário padrão embutido na própria policy), sem necessidade de distinguir público/privado nem checar `is_admin()`.
- Mais simples de manter quando não há necessidade real de múltiplos usuários ou de papéis diferenciados — evita a complexidade de admin/regular do padrão multi-tenant acima.

---

## 5. Autenticação no frontend

- `AuthContext` (`src/contexts/AuthContext.tsx`): expõe `{ user, loading }`, sincroniza com `supabase.auth.getSession()` e `onAuthStateChange`.
- `PrivateRoute` (`src/components/PrivateRoute.tsx`): redireciona para uma rota pública (ex: `/admin` como tela de login) se não houver `user`; não renderiza nada enquanto `loading`.
- Rotas administrativas/de gerenciamento sempre envolvidas por `<PrivateRoute>` em `main.tsx`.
- Se o projeto for **privado**, considerar envolver *todas* as rotas (não só admin) em `PrivateRoute`.

---

## 6. Camada de serviço

- Um único arquivo `src/services/<prefixo>Service.ts` concentra todo acesso ao Supabase.
- Separar em dois objetos exportados:
  - `<prefixo>Service` — chamadas de leitura pública (RPCs `_load_*`).
  - `<prefixo>AdminService` — chamadas de escrita/admin.
- Todo método: `const { data, error } = await supabase.rpc('<rpc_name>', { p_param: valor }); if (error) throw error; return data`.
- Tipos de retorno de cada RPC espelhados em `src/types/<prefixo>.ts`.

---

## 7. Regras gerais de queries e diagnóstico

- Se a entidade-raiz do domínio (equivalente a "torneio" no APT) existir, **toda query em tabelas derivadas deve filtrar por ela** — omitir o filtro mistura dados de instâncias diferentes e pode gerar falsos positivos (ex: "duplicatas" que na verdade são registros de instâncias diferentes).
- **Antes de reportar qualquer anomalia nos dados ao usuário, revisar a própria query** — um filtro faltando pode levar a diagnóstico errado e a ações destrutivas desnecessárias em produção.
- Nunca recomendar apagar/corrigir dados de produção sem certeza do diagnóstico.

---

## 8. Convenções de UI

- Componentes de layout reutilizados: `Nav` (topo, recebe `title` e opcionalmente repassa dados carregados para a página via callback), `SubNav`, `Toast` (feedback de sucesso/erro).
- Páginas administrativas: `Nav` com título da página + card branco `bg-white p-6 rounded-lg shadow` + `Toast`.
- Tema de cores via classes Tailwind customizadas com o prefixo do projeto (ex: `bg-apt-100`, `text-apt-900`), do mais claro ao mais escuro.
- Ícones centralizados em `src/utils/icons.tsx` como um mapa nomeado, evitando imports diretos de `lucide-react` espalhados pelo código.

---

## 9. Configuração e deploy

- `vercel.json`: rewrite `"/(.*)" → "/index.html"` para SPA.
- `.mcp.json`: configura o MCP server do Supabase (`@supabase/mcp-server-supabase`) com `--project-ref` e token de acesso — **nunca commitar este arquivo** (ver `.gitignore`).
- PWA (se aplicável): `sw.js` registrado em `main.tsx` só se `'serviceWorker' in navigator`; ícones em `public/icons/`.

---

## 10. Configuração do Git local

- Um repositório por projeto, remoto único (`origin`) apontando para o GitHub (ex: `github.com/<org>/<repo>.git`), branch principal `main` com tracking configurado (`branch.main.remote=origin`, `branch.main.merge=refs/heads/main`).
- `.gitignore` de referência — nunca commitar:
  ```
  /_ref                        # material de referência local (design, ícones brutos etc.)
  .DS_Store
  /node_modules
  /.claude/settings.local.json # permissões locais do Claude Code, específicas da máquina
  /.mcp.json                   # contém token de acesso do Supabase
  /.env                        # variáveis de ambiente/segredos
  /CLAUDE.md                   # instruções do projeto para o Claude — mantidas locais, não versionadas
  ```
- **`CLAUDE.md` não é versionado** neste padrão — ele existe localmente para orientar o Claude Code, mas não vai para o repositório remoto. Se o novo projeto quiser compartilhar essas instruções com a equipe via repo, isso é uma decisão explícita a tomar (contraria o padrão do APT).
- Nenhum git hook customizado configurado — commits e pushes seguem o fluxo padrão do Git sem automação local.

### Claude e o Git — regra dura

- **Claude nunca executa `git add`, `git commit`, `git push` ou qualquer operação que modifique o repositório.** Isso é proibição absoluta, não uma preferência — o usuário gerencia o git independentemente.
- Claude **pode** usar git para leitura (`git log`, `git diff`, `git status`, `git show`) livremente, para planejamento e diagnóstico de bugs.
- **Mensagem de commit sempre proativa**: ao final de qualquer turno em que Claude tenha usado Edit/Write em arquivo rastreado pelo git, rodar `git status --porcelain` antes de fechar a resposta — se houver qualquer arquivo modificado/não commitado, a resposta final deve incluir a mensagem de commit sugerida, mesmo que o turno tenha sido um fix rápido ou tenha terminado com outro foco. Nunca esperar o usuário perguntar.
- Nunca `--no-verify`, nunca force-push, nunca pular hooks — sem pedido explícito do usuário.

---

## 11. Fluxo de trabalho com o Claude

Esta seção é o que mais importa replicar entre projetos — é como o Claude deve se comportar trabalhando com este usuário especificamente, não uma convenção técnica genérica.

### Comunicação

- O usuário é **PM, não desenvolvedor** — ele entende código na maioria das vezes, mas a explicação deve vir primeiro, em linguagem natural/de produto. Código só como suporte para ilustrar um detalhe específico que não dá para descrever de outra forma.
- Ao apresentar mudanças, análises ou soluções: descrever o que acontece e por quê antes de qualquer bloco de código.

### As 3 etapas obrigatórias (Presentation → Approval → Execution)

- **Toda mudança** — código, migration, RPC, dado em produção — segue: (1) Claude apresenta o problema e a solução proposta com análise de impacto; (2) usuário aprova explicitamente ("pode", "ok" ou equivalente); (3) só então Claude executa.
- Isso vale **mesmo no meio de uma sessão de debugging**, depois de já ter achado a causa raiz — achar o bug não é autorização para corrigi-lo.
- **Sub-decisões não substituem a aprovação final.** Respostas a perguntas pontuais (via AskUserQuestion, ex: "qual cor usar?", "estilo A ou B?") resolvem só aquele detalhe — sempre fazer uma apresentação final consolidada do plano completo e aguardar aprovação explícita antes de tocar em qualquer arquivo.
- Mecanismos de acompanhamento de progresso (`/goal`, stop hooks) **não são permissão para pular a aprovação** — se um hook pressionar para avançar, explicar ao usuário que está aguardando aprovação, não pular a etapa.
- Leituras e investigação podem prosseguir livremente sem plano prévio; só mudanças de estado exigem aprovação.

### Migrations e schema

- **Sempre atualizar o arquivo `.sql` local (`.supabase/functions/` ou `.supabase/tables/`) antes de aplicar via Supabase MCP** — nunca o inverso. O arquivo local é a fonte de verdade; aplicar via MCP sem atualizar o local deixa o repositório desatualizado.
- **`DROP`/`TRUNCATE`**: Claude nunca roda por conta própria, nem em objeto de teste/throwaway. Quando for necessário (ex: trocar assinatura de função, que exige DROP+CREATE porque o PostgREST não escolhe entre duas funções de mesmo nome), mostrar o script completo, avisar claramente que contém DROP e em quê, e só aplicar com o "ok" explícito do usuário.
- **SQL de execução única nunca vira função no banco.** Recálculo, correção pontual de dados, migração one-off: executar direto via MCP (`execute_sql`) ou apresentar o SQL no chat — nunca `CREATE FUNCTION`. Só criar função quando ela for de fato chamada pelo frontend, por cron ou por trigger. Funções one-shot poluem o schema com objetos que nunca serão reutilizados.
- **Nunca criar arquivos temporários (`/tmp` ou scratch) para SQL ou scripts de fix.** Executar diretamente via MCP ou apresentar o bloco no chat.

### Operações em lote sobre dados

- **Sempre pilotar com uma única entidade representativa antes de rodar em todas** — validar o resultado numérico, só então expandir para o lote completo. Não pular a validação mesmo que o código pareça correto.
- Se o batch só altera colunas que não afetam tabelas de cache/derivadas mantidas por trigger (ex: recálculo que só muda `total_quotas`/`quota_value` mas não `balance`), desabilitar os triggers da tabela dentro da transação com `SET LOCAL session_replication_role = 'replica'` — evita overhead desnecessário de triggers que recalculariam algo que não mudou. `LOCAL` garante retorno ao normal automaticamente ao fim da transação.
- Blocos `DO $$` que iteram sobre séries temporais carregando estado anterior (ex: `v_prev_q`) devem tratar explicitamente casos de "re-seed" (ex: resgate total seguido de novo aporte no mesmo id) como primeira condição de cada iteração — o bug de propagar zero para frente só aparece na verificação pós-execução, não no código em si.

### Diagnóstico de dados

- **Nunca afirmar ou assumir valores, convenções de sinal, ou comportamento de dados sem antes consultar o banco.** Rodar uma query de diagnóstico primeiro e apresentar o que os dados mostram, não o que se acha que mostram.
- Ao investigar uma anomalia nos dados, **revisar a própria query antes de reportar ao usuário** — um filtro faltando (ex: falta de filtro pela entidade-raiz do domínio) pode gerar falso positivo e levar a uma correção destrutiva desnecessária em produção.

### Verificação sem servidor local

- O usuário **não roda `npm run dev`** — trabalha só com produção (deploy via Vercel). Verificação visual de frontend acontece depois do deploy, em produção, não localmente.
- Para validar mudanças de frontend, usar `npx tsc --noEmit` (type-check). Se o projeto usar `npm run build` como passo de verificação, **apagar a pasta `dist/` gerada (`rm -rf dist`) imediatamente depois** de confirmar que o build passou — não deixar artefato de build para trás.

### Organização de arquivos

- **Não criar pastas novas sem necessidade.** Antes de `mkdir`, verificar se já existe uma pasta razoável no projeto para o arquivo e usá-la diretamente.

---

## 12. Checklist de setup inicial

- [ ] Definir prefixo do projeto e confirmar público vs. privado, ou multi-tenant vs. single-user (seção 0 / seção 4)
- [ ] Criar repo Git + `.gitignore` (seção 10)
- [ ] `npm create vite@latest` com template React+TS, instalar Tailwind, React Router, `@supabase/supabase-js`, `lucide-react`
- [ ] Configurar `vite.config.ts` (alias `@`, manualChunks) e `tailwind.config.js` (cores do prefixo)
- [ ] Habilitar TypeScript estrito (`strict: true`), banir `any` desde o primeiro commit
- [ ] Criar projeto no Supabase, configurar `.mcp.json` (não commitar)
- [ ] Criar tabela `<prefixo>_profiles` + trigger `handle_new_user` + função `<prefixo>_is_admin()` (multi-tenant) **ou** função `<prefixo>_fnc_load_defaultuser()` (single-user)
- [ ] Definir RLS de cada tabela conforme o modelo escolhido
- [ ] Criar `AuthContext` + `PrivateRoute`
- [ ] Criar `<prefixo>Service.ts` com split leitura/admin, todo acesso ao Supabase via RPC (nunca REST direto em tabela)
- [ ] Criar componentes base: `Nav`, `SubNav`, `Toast`
- [ ] Configurar `vercel.json` para rewrite de SPA
- [ ] Criar `CLAUDE.md` local (não versionado) com as convenções específicas do novo projeto, incluindo a seção 11 deste guia (fluxo de trabalho com o Claude)
