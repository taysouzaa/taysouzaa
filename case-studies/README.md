# Case Studies

Estudos de caso dos sistemas que desenvolvi e mantenho em produção. A maior parte vive em repositório privado de cliente; aqui está a engenharia — o problema, as decisões, os trade-offs e o que eu faria diferente.

**Não contêm** código proprietário, credenciais, endpoints internos, identificadores de infraestrutura ou dado de cliente.

| Projeto | O que resolve | Stack |
|---|---|---|
| [**Cockpit Comercial**](cockpit-comercial.md) | CRM que substituiu a operação comercial em planilhas. Autorização no banco via RLS, não no front. | React · TypeScript · Supabase · n8n |
| [**SPI**](spi.md) | SaaS que padroniza imagens para 6 marketplaces. Processamento no navegador — a imagem original nunca sai do cliente. | React · Node · Prisma · AWS |
| [**Cobrança Recorrente**](cobranca-recorrente.md) | Venda vira assinatura ativa sem passo manual. Webhook idempotente que confirma o pagamento na fonte. | n8n · Pagar.me · Notion · Vercel |
| [**Captação de Leads**](captacao-de-leads.md) | Landing pages num ponto único de entrada, com dedup contra o CRM e qualificação antes do envio. | HTML · CSS · JS · n8n · Supabase |
| [**Calculadoras**](calculadoras.md) | Precificação, margem de contribuição e ROAS de equilíbrio, com histórico entre dispositivos. | React · TypeScript · AWS |
| [**Site Institucional**](site-institucional.md) | Site e blog com captação via proxy same-origin, mantendo credencial fora do navegador. | HTML · CSS · JS · PHP · Apache |

## Código público

O SPI tem versão pública com o código que roda em produção: **[spi-showcase](https://github.com/taysouzaa/spi-showcase)**.

## Um padrão que atravessa os projetos

Os sistemas aqui têm em comum a substituição de operação manual por processo verificável — e, nos casos em que houve incidente, a correção não foi só apagar o erro, foi tornar o erro **visível na próxima vez**. O caso mais claro está em [Cobrança Recorrente](cobranca-recorrente.md#2-erro-engolido-é-pior-que-erro-barulhento).
