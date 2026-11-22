# ADR-007: Snapshot do payload do evento na inserção da outbox

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-004

## Contexto

Com o outbox decidido (ADR-001), ficou a dúvida de modelagem: a linha da outbox guarda o payload já renderizado, ou guarda apenas o order_id e monta o payload na hora do envio [09:51] Bruno. A escolha afeta a fidelidade do evento: se o payload é montado no envio, e o pedido mudar de status de novo entre a gravação e a entrega, o evento pode refletir um estado que não era o da mudança que o originou [09:52] Larissa. Junto dessa dúvida, definiu-se o identificador da linha da outbox: UUID, seguindo o padrão do projeto, em que todas as chaves primárias são UUID [09:51] Diego/Larissa.

## Direcionadores da Decisão

- O evento deve refletir o estado do pedido no momento exato da mudança de status.
- A dedup confiável pelo event_id (ADR-004) depende de payload estável entre tentativas.
- O worker não deveria precisar de consulta adicional ao pedido para entregar.
- Seguir o padrão de identificadores UUID do projeto.

## Alternativas Consideradas

1. **Snapshot na inserção**: renderizar o payload completo do evento no momento da gravação na outbox.
2. **Renderização no envio**: guardar apenas referências (order_id e status) e montar o payload quando o worker for entregar.

## Decisão

Opção 1. O payload do evento é renderizado e gravado pronto na inserção da outbox, dentro da transação da mudança de status [09:52] Larissa, com concordância explícita de Diego (snapshot na inserção) e Bruno [09:52]. Assim, mesmo que o pedido mude depois, o evento entregue reflete o estado de quando o status mudou [09:52] Larissa. O identificador da linha é UUID, como no resto do projeto [09:51] Larissa.

## Prós e Contras das Opções

**Snapshot na inserção**

- Prós: evento fiel ao momento da mudança; a entrega não depende de consulta extra ao pedido; o replay da DLQ reproduz exatamente o evento original.
- Contras: a linha da outbox fica maior, pois carrega o payload completo; o volume da tabela pesa mais, justamente quando o arquivamento já ficou fora do escopo [09:08] Diego.

**Renderização no envio**

- Prós: linha menor; payload sempre fresco no momento da entrega.
- Contras: se o pedido mudou entre a gravação e o envio, o evento reflete estado posterior ao da mudança, o caso esquisito apontado na discussão [09:51] Bruno, [09:52] Larissa; exige consulta ao pedido na entrega; rejeitada.

## Consequências

Positivas: o contrato do evento é determinístico e auditável (o que está na outbox é o que foi entregue); retry e replay reenviam exatamente o mesmo payload, reforçando o dedup por event_id (ADR-004); o worker não monta payload.

Negativas: a tabela da outbox cresce mais por linha; o limite de 64 KB do payload do evento passa a ser validado já na inserção, quando o evento é gravado.

O trade-off assumido: armazenar mais bytes por evento em troca de fidelidade ao momento da mudança e simplicidade do worker.

## Referências

- `src/modules/orders/order.service.ts` (transação em que o snapshot é gravado)
- `prisma/schema.prisma` (modelagem da outbox com payload e id UUID)
- `TRANSCRICAO.md` [09:51]-[09:52] (discussão e decisão)
