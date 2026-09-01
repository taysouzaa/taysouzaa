# Captação e Qualificação de Leads

Landing pages de campanha e o site institucional alimentando um ponto único de entrada, com deduplicação contra o CRM e qualificação antes do envio.

| | |
|---|---|
| **Stack** | HTML · CSS · JavaScript · n8n · Google Sheets · Supabase |
| **Escala** | 9 landing pages · 3 páginas de confirmação · 8 páginas + site integrados ao endpoint · 3 destinos por lead |
| **Estado** | Em produção |

---

## O problema

Campanhas pagas e orgânicas para produtos diferentes — tributação, Product Ads, Shopee Ads, diagnóstico, imersão presencial, consultoria. Cada uma com sua página.

Se cada página falar com o CRM do seu jeito, aparecem três problemas: o mesmo lead entra várias vezes por campanhas diferentes, ninguém sabe de qual anúncio ele veio, e o time comercial recebe uma fila sem prioridade.

## Arquitetura

```
9 landing pages ─┐
site institucional ─┴─▶ POST único ─▶ responde ok (~0,2s)
                                          │
                                   processa depois
                                          │
                        ┌─────────────────┼─────────────────┐
                   normaliza          deduplica         distribui
                (regras de negócio)  (contra o CRM)   (3 destinos)
```

## Decisões

### Ponto único de entrada

Todas as páginas postam no mesmo endpoint. A regra de negócio existe em **um** lugar, não em dez — mudar a definição de lead qualificado é uma alteração, não nove.

O endpoint responde `ok` em cerca de **0,2s** e processa depois. A página não fica esperando o CRM, a planilha e o banco responderem para liberar a tela — o que importa numa landing page de tráfego pago, onde cada segundo de espera é abandono.

### Qualificação calculada antes do envio

O lead é classificado como MQL ainda no navegador, a partir das respostas de qualificação. No produto de tributação, a regra é explícita:

> `MQL = Sim` quando medalha ≠ sem_medalha **e** regime ≠ MEI

Quem está fora do perfil entra marcado, não descartado — o registro continua existindo para análise de campanha.

O comercial recebe fila priorizada em vez de lista crua. É a diferença entre "chegaram 40 leads" e "chegaram 40, 12 valem ligação hoje".

### Atribuição first-touch

As UTMs — `channel`, `source`, `medium`, `campaign`, `content`, `referrer`, `page_url` — são capturadas na primeira visita e **persistidas no navegador**, não lidas no momento do submit.

Lead que chega pelo anúncio, sai para pesquisar e volta digitando o domínio continua atribuído ao anúncio — que foi o que realmente pagou por ele. Sem isso, a campanha paga aparece como tráfego direto no relatório e o cálculo de retorno fica errado para baixo.

### Fila de reenvio no cliente

Conexão móvel cai no meio do submit. Em vez de perder o lead, o envio fica numa fila no navegador e tenta de novo.

### Deduplicação contra o CRM

Antes de criar, o fluxo verifica se o lead já existe e trata reengajamento **como reengajamento** — não como lead novo. Isso preserva o histórico e evita que a mesma pessoa seja distribuída duas vezes para closers diferentes, que é o cenário que gera atrito interno.

## Uma refatoração que valeu

Dois nós do fluxo carregavam a mesma biblioteca de normalização: **337 das 437 linhas eram cópia literal**.

Toda mudança de regra precisava ser feita nos dois lugares. Na primeira vez que alguém esquecesse metade, os dois passariam a discordar em silêncio — normalizando o mesmo lead de formas diferentes conforme o caminho.

A separação ficou por responsabilidade:

- um nó virou **dono das regras de negócio** e passa a emitir a linha do CRM pronta;
- o outro ficou só com o que **depende de consultar o CRM**.

Sobrou uma função em comum, num bloco marcado e verificado por script.

O que tornou a refatoração segura foi a verificação: **126 casos num arnês local mais 5 execuções reais de produção, com zero diferença de comportamento**. Refatorar sem esse passo é reescrever e torcer.

## Decisões de front que parecem detalhe

**Fontes e ícones auto-hospedados.** A tipografia variável vai em WOFF2 no repositório (47 KB) e os ícones são um subset gerado (5,5 KB), em vez de CDN.

Landing page de tráfego pago vive de tempo de carregamento, e cada requisição a terceiro é uma chance de atraso, de bloqueio por extensão ou de falha fora do seu controle.

O custo é real e está documentado: adicionar um ícone novo exige **regerar o subset**. Otimização que ninguém documenta vira armadilha para quem mexer depois.

## O que eu faria diferente

**A regra de MQL vive no navegador.** Isso dá resposta imediata e evita ida ao servidor, mas significa que a definição de lead qualificado está replicada em cada página — exatamente o problema que o ponto único de entrada resolveu do lado do servidor.

Calcular no endpoint, com as páginas apenas enviando as respostas cruas, centralizaria a regra. O custo seria perder a marcação instantânea; o ganho seria mudar o critério em um lugar em vez de nove.
