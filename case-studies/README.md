# Case Studies

Estudos de caso dos sistemas que desenvolvo e mantenho em produção.

A maior parte do código vive em repositório privado de cliente. O que está aqui é a engenharia: o problema, a decisão que define cada projeto, os trade-offs assumidos e o que eu faria diferente hoje.

**Não contêm** código proprietário, credenciais, endpoints internos, identificadores de infraestrutura nem dado de cliente.

---

## Os projetos

| Projeto | A decisão que o define | Stack |
|---|---|---|
| [**Cockpit Comercial**](cockpit-comercial.md) | Autorização no banco via RLS, não no front — a regra vale para qualquer cliente que se conecte, não só para a tela | React · TypeScript · Supabase · n8n |
| [**SPI**](spi.md) | Processamento no navegador via Canvas API — a imagem original nunca sai da máquina do cliente | React · Node · Prisma · AWS |
| [**Cobrança Recorrente**](cobranca-recorrente.md) | O webhook é gatilho, não fonte de verdade: o pagamento é confirmado na API do gateway antes de criar a assinatura | n8n · Pagar.me · Notion · Vercel |
| [**Captação de Leads**](captacao-de-leads.md) | Ponto único de entrada para 9 páginas — a regra de negócio existe em um lugar, não em nove | HTML · CSS · JS · n8n · Supabase |
| [**Calculadoras**](calculadoras.md) | Cálculo puro separado da tela, para que a fórmula possa ser conferida por quem entende do negócio | React · TypeScript · AWS |
| [**Site Institucional**](site-institucional.md) | Proxy same-origin — chave em código de cliente é chave pública | HTML · CSS · JS · PHP · Apache |

## Código público

O SPI tem versão pública com o mesmo código que roda em produção: **[spi-showcase](https://github.com/taysouzaa/spi-showcase)** — React + Node + Prisma + AWS, com CI de typecheck, testes e build.

---

## Dois padrões que atravessam os projetos

**Substituir operação manual por processo verificável.** Não é automatizar por automatizar: cada sistema aqui nasceu de uma etapa que dependia de alguém lembrar. Planilha que qualquer um edita, cobrança que alguém precisa anotar, lead que chega sem origem. O ganho não é velocidade, é a operação parar de depender de memória.

**Tornar o erro visível na próxima vez.** Nos casos em que houve incidente, a correção não parou em apagar o sintoma:

- Em [Cobrança Recorrente](cobranca-recorrente.md#2-erro-engolido-é-pior-que-erro-barulhento), 15 dias de gravações perdidas apareciam como execuções bem-sucedidas. A correção foi fazer a falha interromper o fluxo — assim ela fica vermelha *e* o gateway reenvia sozinho.
- No [Cockpit](cockpit-comercial.md#um-bug-que-virou-método), desativar um perfil não cortava o acesso. A correção virou método: para cada regra que nega, existe uma contraprova que exige.
- Nas [Calculadoras](calculadoras.md#um-bug-que-só-o-typecheck-encontrou), uma função nunca importada passava pelo build. A correção foi o CI que faltava.

Verde que não prova nada é pior que vermelho.
