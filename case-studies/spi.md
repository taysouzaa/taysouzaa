# SPI — Sistema de Processamento de Imagens

SaaS que padroniza imagens de produto para 6 marketplaces.

**Stack:** React · Vite · TypeScript · Node + Express · Prisma · PostgreSQL · AWS (S3, EC2, Amplify)
**Estado:** em produção · **Código público:** [spi-showcase](https://github.com/taysouzaa/spi-showcase)

---

## O problema

Cada marketplace exige a imagem do produto num formato próprio — Amazon 1000×1000, Shein 1500×2000, TikTok Shop 800×800, todos com fundo branco. Um seller que anuncia em seis canais refaz a mesma foto seis vezes, à mão, no editor.

## A decisão que define o projeto

**O processamento acontece no navegador, não no servidor.**

A Canvas API redimensiona, centraliza e aplica o fundo branco na máquina do usuário. Só o resultado final sobe para o S3 — a imagem original nunca sai do computador dele.

Três consequências, e uma delas não é técnica:

1. **Custo de servidor vira trabalho do cliente.** Processar imagem é caro em CPU e memória; numa instância pequena, um lote de 50 uploads derruba o serviço. No navegador, cada usuário traz o próprio processador.
2. **Some o upload da imagem bruta.** O arquivo que trafega é o processado, tipicamente menor.
3. **Resolve uma objeção comercial.** Seller não gosta de mandar foto de produto que ainda não foi lançado para servidor de terceiro. Aqui a resposta é verificável: a imagem não é enviada.

O trade-off: fica dependente do navegador do usuário. Máquina fraca processa lote grande devagar, e não dá para oferecer processamento agendado sem repensar a arquitetura.

O backend cuida só do que exige confiança: autenticação, histórico e gestão de usuários.

## Decisões de backend

**Autenticação sem senha.** Login por e-mail + telefone. O modelo `User` não tem campo de senha — não há hash para vazar, nem fluxo de recuperação para atacar. Sessão em JWT com access de 15 minutos e refresh de 30 dias.

**Bucket privado, URL pré-assinada.** O S3 nunca é público. Para baixar do histórico, o backend assina uma URL temporária e o navegador busca o objeto direto no S3 — o binário não passa pela instância EC2, o que tira a transferência do caminho do servidor.

**Limite de concorrência com fila.** Upload passa por um limitador com fila limitada e timeout por item. Fila cheia responde 503 na hora, em vez de acumular indefinidamente e derrubar a instância. O gargalo real não é a cota do S3 — é a máquina segurando file descriptor e memória por upload em voo.

**Modo degradado no histórico.** Se o Postgres cai, o histórico responde a partir de um arquivo local. Não substitui o banco — dado se perde em ambiente efêmero — mas mantém a tela legível durante uma indisponibilidade.

## Qualidade

CI a cada push e PR: testes e build do backend, typecheck e build do front. 16 casos cobrindo o que quebra em silêncio — registro duplicado, credencial errada, banco indisponível, usuário bloqueado.

O middleware de autenticação é **fail-closed**: quando o banco não responde, ele retorna 503 em vez de deixar passar. É o comportamento que um teste precisa travar, porque a tentação de "liberar enquanto o banco não volta" aparece sempre.

## Números

| | |
|---|---|
| Marketplaces suportados | 6 |
| Linhas de código | ~7.700 |
| Casos de teste | 16 |
| CI | typecheck + testes + build, verde |
