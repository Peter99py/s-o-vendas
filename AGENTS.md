# AGENTS.md — S&O Vendas

Guia para agentes de IA (e humanos) que trabalham neste projeto. Objetivo: dar contexto mínimo suficiente, orientar mudanças corretas e economizar tokens.

Leia este arquivo por completo antes de agir. Ele resume o essencial; para detalhes, consulte o README do repositório alvo.

## Modo passivo (padrão sempre)

- Por padrão o agente NÃO altera código. Analise, explique e proponha (plano/diff conceitual) e aguarde a decisão do usuário.
- Só escreva/edite/apague arquivos, crie migrations, ou rode comandos que modifiquem o projeto quando o usuário pedir explicitamente a execução (ex.: "implemente", "aplique", "faça a mudança", "pode executar", "corrija").
- Pedidos de análise não autorizam modificações: "como funciona", "onde está", "o que você acha", "por que quebrou", "explique" e similares são apenas leitura/consulta.
- Em caso de dúvida sobre a autorização, pergunte antes de editar qualquer arquivo.

## 0. Escopo e mapa do projeto

A raiz do workspace (s-o-vendas/) não é um repositório Git. Ela contém dois repositórios Git independentes:

- s-o-vendas-backend/ — API + gestão do PDV offline-first. Django 5.2 LTS + Django Ninja; SQLite (dev) / PostgreSQL (prod).
- s-o-vendas-frontend/ — PWA offline-first do PDV. React 19 + TypeScript + Vite 7 + Tailwind 4 (pnpm workspace).

Commits, branches e git status são por repositório. Sempre rode cd para o repo alvo antes de operações Git.

### Backend (s-o-vendas-backend/)

```
config/    settings, urls, api (Ninja), middleware CSP/CORS, wsgi/asgi
apps/
  core/       lojas, usuários/perfis, hierarquia gerente→vendedor (M2M Gerencia),
              clientes, fornecedores, configuração
  catalog/    produtos, categorias, unidades, preços por local, ledger de estoque
  sales/      vendas, itens, parcelas, recebimentos, devoluções/estornos,
              transferências, ajustes, crédito de cliente
  finance/    despesas / categorias de despesa
  purchases/  compras / entrada de mercadorias
  reports/    relatórios agregados
  sync/       dispositivos, cursors, log de sync (idempotência por UUID)
```

Cada app segue o padrão: models.py, api.py (router Ninja), services.py (regra de negócio), admin.py, tests.py, migrations/.

### Frontend (s-o-vendas-frontend/)

Workspace pnpm: packages/* + apps/*.

```
packages/ui/     @sovendas/ui — design system; componentes em src/components/ui/,
                 reexportados por src/index.ts; tokens em src/tokens.css
apps/web/src/
  app/       router (rotas + guard por papel), providers, auth (zustand), stores de tema/config
  features/  auth, venda, catalogo, clientes, vendas, compras, despesas,
             devolucao, relatorios, admin, impressao, download, styleguide
  api/       client (fetch mesma origem, sessão) + endpoints
  db/        camada offline (Dexie): schema, outbox, purge, sync
  lib/       utilitários (pix, uuid)
```

## 1. Economia de tokens (obrigatório)

1. Não releia arquivos após write_file/edit_file — a chamada falha se não funcionar.
2. Não cole arquivos inteiros, READMEs ou logs na resposta. Referencie caminhos; descreva a mudança como um diff conceitual.
3. Localize antes de ler: use grep/find_path para achar o símbolo e read_file com start_line/end_line para ler só o trecho relevante.
4. Não despeje arquivos grandes nem saída de build/testes. Filtre/limite com head_lines/tail_lines e resuma. Para logs volumosos, delegue a um subagente que devolva só o resumo com as linhas de erro relevantes.
5. Paralelize chamadas de ferramenta independentes numa única resposta.
6. Sem preâmbulo verboso: no máximo 1–2 frases antes de um grupo de ações.
7. Não repita o que já está no contexto; evite "aqui está o arquivo completo".
8. Confirme no código antes de afirmar comportamento; marque inferências como inferências.
9. Prefira os alvos make existentes a inventar comandos.
10. Termine com um resumo curto: o que mudou, arquivos, validação executada (ou por que não).
11. Formatação de .md enxuta: escreva documentos em texto simples. Evite emojis, negrito decorativo, tabelas, linhas horizontais, badges, blockquotes e caixas. Prefira listas e parágrafos diretos. Use crase e code fences apenas para identificadores, caminhos e comandos. A formatação existe para organizar, não para decorar — visual pesado consome tokens sem agregar.

## 2. Regras de desenvolvimento

### Princípios gerais (valem para os dois repos)

- Modo passivo (padrão). Só modifique o código quando o usuário pedir explicitamente — ver a seção "Modo passivo" no topo.
- Mudança mínima e focada. Não refatore além do pedido; siga o estilo existente.
- Não commite nem crie branches sem pedido explícito. Não desfaça trabalho do usuário.
- Identificadores em inglês; textos de UI em pt-BR.
- Dependências fixadas: requirements.txt usa ==; package.json usa versões exatas. Não faça upgrade oportunista — atualize conscientemente.

### Comentários no código

- Comente apenas o que o código não revela: intenção, regra de negócio, restrição externa, decisão de projeto, tradeoff. Explique o porquê, não o quê.
- Não comente o óbvio nem reapresente a linha em palavras (ex.: "// incrementa i"). Nomes claros dispensam comentário.
- Prefira nomes e estrutura a comentários explicativos; extraia uma função em vez de descrever o bloco.
- Não deixe código comentado "para depois" nem TODO/FIXME órfão sem dono e motivo.
- Comentário desatualizado é pior que nenhum: ao mudar o comportamento, atualize ou remova o comentário junto.
- Mantenha o idioma do comentário consistente com o arquivo (predominante em inglês no backend; siga o padrão local).

### Invariantes de domínio (backend)

- Append-only: ledgers (movimentos de estoque, recebimentos, estornos, devoluções, créditos) nunca são editados/excluídos — o estado muda por novo registro. O Admin bloqueia exclusão desses modelos.
- Escopo por papel aplicado na API e no Admin: vendedor vê só o próprio; gerente vê subordinados (M2M Gerencia); admin vê tudo.
- Regra de negócio em services.py de cada app (ex.: finalizar_venda, cancelar_venda, registrar_devolucao, registrar_recebimento, registrar_transferencia, registrar_ajuste, saldo_atual). Não coloque lógica de negócio nas views/api.py.
- Dinheiro com Decimal no backend — nunca float.
- UUIDs são as chaves de idempotência do sync (POST/GET /api/sync, dedupe por UUID).
- Toda mudança de modelo exige migration versionada (make makemigrations). Nunca edite migrations já aplicadas.

### Convenções do frontend

- Nova entidade do backend: espelhar em db/schema.ts e mapear em db/sync.ts (TABELAS).
- Páginas leem do Dexie, nunca do servidor direto; escrita sempre local + outbox.
- Valores monetários em centavos.
- Sem nenhuma URL externa em código/estilos/HTML (CSP 'self' em produção): fontes, ícones e imagens são bundlados.
- Design system antes de visual à mão: use componentes de packages/ui (shadcn/Radix reexportados por @sovendas/ui) e os tokens de tokens.css em vez de criar classes soltas. Variantes via class-variance-authority.

## 3. Validação e Definition of Done

Rode a validação o mais específico possível ao que mudou e depois amplie. Reporte falhas com o comando e o erro relevante.

### Backend (s-o-vendas-backend/)

```sh
make check
.venv/bin/ruff check . --exclude .venv
.venv/bin/python manage.py makemigrations --check --dry-run
.venv/bin/python manage.py test
```

### Frontend (s-o-vendas-frontend/)

```sh
make typecheck
make build
```

### Código morto e duplicado (após cada execução)

Obrigatório antes de considerar a tarefa concluída:

- Código morto
  - Backend: ruff check sinaliza import não usado (F401), variável local não usada (F841), redefinição (F811) etc. Não silencie com # noqa sem justificativa.
  - Frontend: make typecheck já reprova locais e parâmetros não usados (noUnusedLocals/noUnusedParameters).
  - Remova imports, variáveis, funções, arquivos e exports órfãos que você criou e não usou.
- Código duplicado
  - Antes de criar um helper/componente/serviço, procure o equivalente (grep pelo nome e por sinônimos; confira packages/ui, lib/, services.py). Reutilize ou extraia para um lugar comum em vez de copiar.
  - Ao remover uma função, busque por chamadas remanescentes (grep) para não deixar referência quebrada.
- Migrações: nenhuma migration pendente (makemigrations --check silencioso).

### Definition of Done

1. Mudança implementada e focada no pedido.
2. Lint/typecheck limpos; sem código morto/duplicado introduzido.
3. Migrations criadas quando o modelo mudou.
4. Testes relevantes passando (ou, se não houver cobertura, explique o que foi validado manualmente).
5. Resumo final: arquivos alterados + validação executada.

## 4. Comandos úteis

Backend — alvos do Makefile:

```sh
make install     # cria .venv e instala requirements
make dev         # Postgres em container (host:5433) + runserver 0.0.0.0:8000
make migrate
make makemigrations
make check
make superuser   # cria/reseta superuser do admin (idempotente)
make prod        # build + up -d (db + gunicorn)
```

Sem Docker: python manage.py runserver (cai para SQLite quando não há DATABASE_URL).

Frontend — alvos do Makefile:

```sh
make install     # pnpm install
make dev         # dev server (http://localhost:5173)
make typecheck
make build
make preview
```

## 5. Estado atual validado / armadilhas conhecidas

Verificado nesta revisão — não presuma o contrário:

- .venv (backend) e node_modules (frontend) não estão instalados no checkout atual. Rode make install antes de validar/rodar.
- ruff é citado no README do backend mas NÃO está em requirements.txt. Instale/verifique (ex.: pip install ruff) antes de confiar no passo de lint.
- pnpm não está no PATH neste ambiente. O Makefile do frontend cai no fallback npx --yes pnpm@9 — use os alvos make em vez de chamar pnpm direto.
- Os docs referenciados pelos READMEs (../docs/visao-geral.md, ../docs/modelo-de-dados.md) NÃO existem na árvore. Não conte com eles; a fonte de verdade é o código.
- O README do backend está desatualizado: omite os apps finance, purchases e reports, que existem no código. Confie na lista do config/settings.py / diretório apps/.
- Sem ESLint/Prettier no frontend. A rede de segurança é o tsc (make typecheck) + estilo existente.
- Sem ferramenta de teste no frontend. No backend há tests.py em sales, sync, purchases e reports (Django TestCase).
- make dev do backend exige Docker e rede local para o Postgres na porta host 5433 (a 5432 fica livre para um Postgres local, se houver).
- Deploy: backend via Docker/Gunicorn no Railway; frontend via Docker (nginx) no Railway, com VITE_API_BASE apontando para a URL pública do backend.
