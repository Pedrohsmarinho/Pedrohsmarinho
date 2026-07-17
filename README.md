# Desafio Técnico — Gestão de Pedidos (Claro)

Sistema de gestão de pedidos de e-commerce: API REST em Spring Boot + SPA em
Angular, com observabilidade completa (métricas, logs e traces — Prometheus,
Loki e Tempo, visualizados no Grafana) como diferencial.

## Estrutura do repositório

```
/backend    -> Spring Boot 3.x (Java 17+), API REST + MariaDB
/frontend   -> Angular 17 (standalone components)
/monitoring -> configs de Prometheus/Loki/Promtail/Tempo + dashboards do Grafana
/postman    -> collection + environment do Postman e script de curl
docker-compose.yml
CONTRIBUTING.md -> fluxo de branches (Gitflow) e Conventional Commits
```

## Como executar

### Via Docker Compose (recomendado)

```bash
cp .env.example .env
# edite o .env e defina JWT_SECRET com um valor aleatório, ex:
openssl rand -base64 48

docker compose up --build -d
```

Um único comando sobe tudo: backend, frontend, MariaDB e a stack completa de
observabilidade (Prometheus + Loki + Tempo + Grafana, o "LGTM stack"). O
backend só inicia depois que o MariaDB reporta saudável (`service_healthy`).

Para começar do zero (apaga também os volumes de dados):
```bash
docker compose down -v && docker compose up --build -d
```

| Serviço | URL |
|---|---|
| Frontend | http://localhost:4200 |
| Backend | http://localhost:8080 |
| Swagger UI | http://localhost:8080/swagger-ui/index.html |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (usuário `admin`, senha `admin`) |
| MariaDB | `localhost:3307` (usuário `pedidos_user`, senha `pedidos_pass`, banco `pedidos_db`) |

Um dashboard **"Pedidos API - Visão Geral e Saúde"** já vem provisionado no
Grafana, reunindo cards de negócio, saúde da API (`up`), métricas técnicas
(requisições/s, latência, erros 4xx/5xx) e um painel de logs em tempo real.
Cada log carrega `traceId`/`spanId`, correlacionado automaticamente com o
Tempo (logs ↔ traces nos dois sentidos).

### Local, sem Docker

Backend: requer um MariaDB em `localhost:3306` (schema/usuário no README
completo ou no `docker-compose.yml`).
```bash
cd backend
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```
Frontend:
```bash
cd frontend
npm install
npm start
```

A API sobe em `http://localhost:8080`, o frontend em `http://localhost:4200`.
O schema e o seed inicial (3 pedidos) são criados automaticamente pelo
`DataSeeder` na primeira subida. O fluxo normal é criar sua própria conta em
`http://localhost:4200` (seletor "Criar conta" — nome, email, senha mínima
de 8 caracteres), com login automático após o cadastro.

## Modelo de domínio — Pedido

| Campo         | Tipo               | Observação                               |
|---------------|--------------------|-------------------------------------------|
| `id`          | Long               | gerado pelo banco                          |
| `displayName` | String             | nome do cliente/pedido                     |
| `itens`       | Integer            | quantidade de itens                        |
| `peso`        | Long               | **armazenado sempre em gramas**            |
| `status`      | enum StatusPedido  | `EM_PROCESSAMENTO`, `PAUSADO`, `CANCELADO` |

O frontend converte a exibição para kg e converte de volta para gramas antes
do `POST` (input do usuário em kg, mais natural para e-commerce).

**Transições de status válidas** (`StatusPedido.podeTransicionarPara`):
```
EM_PROCESSAMENTO -> PAUSADO / CANCELADO
PAUSADO          -> CANCELADO / EM_PROCESSAMENTO
CANCELADO        -> EM_PROCESSAMENTO
```
Qualquer transição fora dessa tabela retorna `422`.

**Limite de negócio**: máximo de 5 pedidos simultâneos **por usuário** (não
global) — decisão tomada ao evoluir o desafio original (single-tenant) para
multiusuário. Tentativas acima do limite retornam `422` e ficam logadas em
`WARN`.

## Contrato da API

| Método | Endpoint                    | Descrição                              | Auth | Sucesso | Erros |
|--------|------------------------------|-----------------------------------------|------|---------|-------|
| POST   | `/api/auth/login`            | Autentica o usuário, retorna JWT        | não  | 200     | 400, 401 |
| POST   | `/api/auth/registrar`        | Cadastra um usuário, já retorna JWT     | não  | 201     | 400, 409 |
| GET    | `/api/pedidos`               | Lista os pedidos do usuário autenticado | sim  | 200     | 401 |
| POST   | `/api/pedidos`               | Cadastra um novo pedido                 | sim  | 201     | 400, 401, 422 |
| PATCH  | `/api/pedidos/{id}/status`   | Altera o status de um pedido            | sim  | 200     | 400, 401, 404, 422 |
| DELETE | `/api/pedidos/{id}`          | Exclui um pedido                        | sim  | 204     | 401, 404 |

CORS habilitado apenas para `http://localhost:4200`. Documentação interativa
via Swagger UI (`springdoc-openapi`), gerada automaticamente dos controllers.

**Multi-tenant**: todas as rotas de `/api/pedidos/**` operam exclusivamente
sobre os pedidos do usuário do token (nunca a partir de um `usuarioId` do
corpo/path/query), fechando uma falha de autorização do tipo IDOR. Um `id`
de pedido de outro usuário retorna `404` (não `403`), para não vazar a
existência do recurso a quem está tentando o ataque.

## Observabilidade

- **Prometheus**: métricas técnicas padrão + métricas de negócio customizadas
  (`pedidos_total` como Counter cumulativo, `pedidos_by_status`,
  `pedidos_peso_total_gramas`, `pedidos_itens` como Gauges) — todas globais
  (somam todos os usuários), usadas para saúde/uso agregado do sistema.
- **Tracing** (Micrometer + OTLP → Tempo): não estava nos requisitos, mas
  fecha o tripé métricas + logs + traces, respondendo *por quê* uma
  requisição específica falhou ou foi lenta — sem atrasar o restante do
  escopo obrigatório.
- **Logs** (Promtail → Loki), correlacionados com os traces via `traceId`.

Duas armadilhas de configuração documentadas no histórico: o Tempo v3 mudou
o schema de `ingester`/`compactor` para "live-store"; e o volume do Grafana
precisou ser recriado (`down -v`) ao adicionar Loki/Tempo depois do
Prometheus já estar rodando.

## Decisões técnicas — Backend

- **Java 17 / Spring Boot 3.5**, **MariaDB** como banco principal (H2 só em
  testes) — o enunciado aceitava H2 em memória, mas foi pedida persistência
  real.
- **Regra de transição de status no enum** `StatusPedido`, não no service —
  mantém a regra de negócio junto ao domínio.
- **Exceções de negócio dedicadas** + `@RestControllerAdvice` centralizando
  o mapeamento para os códigos HTTP exigidos, cobrindo inclusive exceções
  técnicas do Spring que sem handler cairiam em 500.
- **Evolução para multiusuário + JWT completo**: entidade `Usuario` (senha
  com hash BCrypt), cadastro via `POST /api/auth/registrar` com login
  automático (retorna JWT), e `POST /api/auth/login` retornando um JWT real.
- **Senha**: mínimo de 8 caracteres (sem exigir maiúscula/número/símbolo),
  combinado com BCrypt.
- **JWT** (`io.jsonwebtoken`, HS256, expiração de 1h): payload só com `sub`
  e `exp`. O secret vem de `JWT_SECRET` sem default no profile principal —
  a aplicação falha na subida se não for definido.
- **Autorização por usuário** em todos os métodos de `PedidoService`
  (`usuarioId` explícito, nunca lido do request).
- **Peso sempre em gramas na API**, conversão para kg só na apresentação.
- **Logs estruturados** via SLF4J/Logback (`INFO`/`WARN`/`ERROR` conforme o
  cenário).
- **Testes**: 83 testes JUnit no total (transições de status, isolamento
  entre usuários, autenticação, segurança via contexto Spring completo,
  busca/paginação, dashboard, validação e cenários de erro/exceção).

### Cobertura de código (JaCoCo)

Sem threshold mínimo configurado ainda (decisão deliberada de olhar o número
real antes de travar uma meta). Evoluiu de **92.8%** (44 testes, caminho
feliz) para **99.4% de linhas** (65 testes, com cenários de erro completos).
Dois gaps reais foram corrigidos no código ao escrever esses testes: rota
inexistente retornava `500` em vez de `404` (`NoResourceFoundException` sem
handler), e corpo malformado (ex. `"peso": "abc"`) retornava `500` em vez de
`400` (`HttpMessageNotReadableException` sem handler).

## Decisões técnicas — Frontend

- **Standalone components** (sem NgModules), padrão recomendado a partir do
  Angular 17.
- **Angular Material** para formulários, tabela, cards e diálogos; **ng2-
  charts + Chart.js** para os gráficos do dashboard.
- **Fallback em LocalStorage**: se o `POST` falhar por indisponibilidade da
  API (status `0`, não erro de negócio), o pedido é salvo localmente e
  mesclado na listagem/dashboard até a API voltar.
- **Peso digitado em kg**, convertido para gramas antes do envio.
- **Login e cadastro na mesma tela**, alternando via `mat-button-toggle-
  group`.
- **Token em `sessionStorage`** (não `localStorage`), para reduzir a janela
  de exposição — um cookie `httpOnly` seria mais seguro, mas exigiria mudar
  o fluxo de CORS/CSRF, fora do escopo aqui.
- **`authGuard`/`authInterceptor` funcionais** (padrão Angular 17):
  verificam expiração do token localmente e reagem a `401` do backend.
- **Identidade visual**: paleta da marca Claro (vermelho `#e4002b` +
  branco/cinza), tipografia Manrope, cores de status como variáveis CSS
  próprias.
- **Dashboard consome `GET /api/dashboard/metricas`** (query autoritativa no
  backend, escopada por usuário) em vez de recontar a lista no navegador ou
  ler direto do `MeterRegistry` (que é global e cumulativo, não serve para
  "quantos pedidos esse usuário tem agora").
- **Estados vazio/carregando/API indisponível** tratados explicitamente em
  vez de mostrar uma tabela vazia sem explicação.
- **Indicador de saúde da API** na toolbar (polling a cada 30s em
  `/actuator/health`), separado do aviso de fallback local da listagem.
- **Filtro, busca, paginação e ordenação resolvidos no backend**
  (`/api/pedidos/busca`), com debounce de 300ms na busca por nome.
- **Polling do dashboard a cada 20s**, complementando o Observable
  compartilhado (cobre mudanças vindas de outra aba/dispositivo).
- **CSS em `rem`** (não `px`) para espaçamento/tipografia/larguras-teto,
  garantindo escala com zoom/fonte do navegador; `px` mantido só para
  detalhes puramente visuais (bordas, sombras, raios).
- **Correções de UX/responsividade**: toolbar quebrando em telas estreitas
  e tabela "empurrando" a página no mobile — ambos corrigidos.
- **`package-lock.json` não versionado**: mitigado fixando versões exatas
  (sem `^`/`~`) em todas as dependências do `package.json`.
- **Testes**: 71 testes Jasmine/Karma cobrindo transição de status,
  fallback local, guard/interceptor, serviços de auth/dashboard/health,
  busca/paginação e os fluxos de login/cadastro/pedido.

### Revisão de qualidade (code review)

Duas convenções estabelecidas: `extrairMensagemErro()` (utilitário único
para mensagens de erro HTTP, usado nas telas que antes reimplementavam a
mesma lógica) e `PedidoService.limiteMaximo$` (fonte única do limite de
pedidos no frontend, populada via API em vez de uma constante estática
desatualizável). Também corrigido: `id` não numérico em rotas de pedido
retornava `500` em vez de `400`/`405`.

## Validação realizada

- **Backend**: bateria de `curl` cobrindo login, registro, CRUD de pedidos,
  limites e autorização; 83 testes JUnit rodando contra H2, com 100% de
  linhas cobertas no `GlobalExceptionHandler`.
- **Frontend**: `ng build` sem erros; 71 testes Jasmine/Karma.
- **Visual**: screenshots reais (Chrome headless) em 1440px e 390px,
  usados para validar as correções de responsividade.

**Recomendação**: mesmo assim, antes de considerar o fluxo 100% validado,
percorra manualmente login/cadastro → dashboard → listagem → cadastro de
pedido → mudança de status → exclusão → logout.

## O que eu faria diferente com mais tempo

- **Refresh token** (com rotação/revogação), evitando reautenticação a cada
  1h sem aumentar a janela de exposição do token.
- **Verificação de email** no cadastro.
- **Cache** em `UsuarioRepository.findByEmailIgnoreCase` e em
  `/api/dashboard/metricas` (não feito agora por não se justificar no
  volume deste desafio).
- **Testes end-to-end de verdade** (Cypress/Playwright), em vez de apenas
  unitários + validação visual manual.
- **CI configurado** (GitHub Actions) rodando os testes e o build do Docker
  a cada push/PR.
- **Internacionalização (i18n)** — hoje todo texto está hardcoded em
  português.
- **Guia de acessibilidade**: faltam `aria-label`s nos botões de ação da
  listagem e um audit de contraste mais completo.
- **Prints/GIF da aplicação e do dashboard Grafana no README.**
