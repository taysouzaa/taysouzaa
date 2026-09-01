# SPI — Sistema de Processamento de Imagens

SaaS que padroniza imagens de produto para 6 marketplaces.

| | |
|---|---|
| **Stack** | React 18 · Vite · TypeScript · Node 20 + Express · Prisma · PostgreSQL (RDS) · AWS (S3, EC2, Amplify) |
| **Escala** | ~7.700 LOC · 16 casos de teste · CI com typecheck, testes e build |
| **Estado** | Em produção · **[código público](https://github.com/taysouzaa/spi-showcase)** |

---

## O problema

Cada marketplace exige a imagem do produto num formato próprio — Amazon 1000×1000, Shopee 1024×1024, Shein 1500×2000 na proporção 3:4, TikTok Shop 800×800 — todos com fundo branco puro. Um seller que anuncia em seis canais refaz a mesma foto seis vezes, à mão, no editor.

## A decisão que define o projeto

**O processamento acontece no navegador, não no servidor.**

A Canvas API redimensiona, centraliza e aplica o fundo branco na máquina do usuário. Só o resultado final sobe para o S3 — a imagem original nunca sai do computador dele.

```
Navegador                              Servidor
─────────────────────────────          ──────────────────────
imagem original
   │
   ├─ Canvas: downscale progressivo
   ├─ letterbox + fundo #FFFFFF
   │
   └─ resultado ────────────────────▶  POST /images/upload
                                          │
                                          ├─ limiter + timeout
                                          ├─ S3 PutObject
                                          └─ Prisma: histórico
```

Três consequências, e a terceira não é técnica:

1. **Custo de servidor vira trabalho do cliente.** Processar imagem é caro em CPU e memória; numa instância pequena, um lote de 50 uploads derruba o serviço. No navegador, cada usuário traz o próprio processador — e o custo de infra não cresce com o uso.
2. **Some o upload da imagem bruta.** O que trafega é o arquivo processado, tipicamente menor. Em conexão móvel isso é a diferença entre usável e não usável.
3. **Resolve uma objeção comercial.** Seller não gosta de mandar foto de produto ainda não lançado para servidor de terceiro. Aqui a resposta é verificável no DevTools: a imagem não é enviada.

**O trade-off:** fica dependente do navegador do usuário. Máquina fraca processa lote grande devagar, e não dá para oferecer processamento agendado sem repensar a arquitetura.

A qualidade do resultado exigiu cuidado específico: **downscale progressivo** em vez de redimensionamento em um passo, mais `imageSmoothingQuality: 'high'`. Reduzir 3000px para 1000px de uma vez produz serrilhado visível — e imagem de produto reprovada por qualidade é o único erro que o cliente percebe.

O backend cuida só do que exige confiança: autenticação, histórico e gestão de usuários.

## Decisões de backend

**Autenticação sem senha.** Login por e-mail + telefone, validados com Zod. O model `User` **não tem campo de senha**: não há hash para vazar, nem fluxo de recuperação para atacar, nem usuário reciclando a senha do e-mail. Sessão em JWT com access de 15 minutos e refresh de 30 dias.

O trade-off é explícito: o fator é o par e-mail + telefone, o que é adequado para uma ferramenta de produtividade B2B com base de usuários conhecida, e não seria para um produto aberto ao público.

**Bucket privado, URL pré-assinada.** O S3 nunca é público. Para baixar do histórico, `GET /history/:id/download-url` devolve uma URL assinada com TTL, e o navegador busca o objeto direto no S3. O binário **não passa pela instância EC2** — tira a transferência do caminho do servidor e mantém o bucket fechado.

**Limite de concorrência com fila.** Upload passa por um limitador com fila limitada e timeout por item; fila cheia responde `503` na hora em vez de acumular. O gargalo real não é cota do S3 — é a `t3.micro` segurando file descriptor e memória por upload em voo. Sem o teto, um pico não degrada: derruba.

**Modo degradado no histórico.** Se o Postgres cai, o histórico responde a partir de um arquivo local. Não substitui o banco — dado se perde em ambiente efêmero, e está documentado como limitação — mas mantém a tela legível durante uma indisponibilidade em vez de mostrar erro.

## Verificação

CI a cada push e PR: testes e build do backend, typecheck e build do front. Os testes usam variáveis fake e Prisma mockado, então **nenhum secret é necessário** no pipeline.

16 casos, escolhidos por onde a falha é silenciosa:

| Suíte | O que protege |
|---|---|
| `auth` | Registro duplicado (409), e-mail e telefone incorretos (401), banco indisponível (503) |
| `middlewares` | Token ausente ou inválido (401), usuário bloqueado (403), e **fail-closed** quando o banco cai |
| `history.pagination` | Teto de `limit` em 100, defaults, cálculo de `totalPages` e `skip` |

O `fail-closed` é o que mais importa: quando o banco não responde, o middleware retorna 503 em vez de deixar passar. É o comportamento que um teste precisa travar, porque a tentação de "libera enquanto o banco não volta" aparece toda vez que há incidente.

## O que eu faria diferente

**O processamento não tem teste.** A lógica de redimensionamento roda no navegador e é verificada visualmente. Extrair o cálculo de dimensões e letterbox para funções puras — separadas do Canvas — permitiria testar as seis regras de marketplace sem browser, e é onde um erro sai caro: imagem fora de especificação é anúncio reprovado.

**`ProcessorView.tsx` tem 767 linhas.** Concentra upload, fila, processamento e download. É o próximo candidato a ser quebrado em hooks.
