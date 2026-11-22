# ADR-004: Entrega at-least-once com X-Event-Id para deduplicação no cliente

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-002, ADR-003

## Contexto

Com outbox e retry (ADR-001, ADR-002), não há coordenação de transação com o endpoint do cliente. Depois de esgotado o timeout de 10 segundos da chamada [09:42] Diego, o worker não consegue distinguir "cliente não recebeu" de "cliente recebeu e a resposta se perdeu"; a retentativa pode gerar entrega duplicada. Duplicatas são inevitáveis em qualquer sistema de notificação com retry.

A reunião fechou o posicionamento: a plataforma garante at-least-once e o cliente precisa estar preparado para receber o mesmo evento mais de uma vez [09:24] Diego. Sofia registrou que isso joga responsabilidade para o cliente [09:25] Sofia; Diego argumentou que é o padrão de mercado, citando Stripe e GitHub, e que exactly-once exigiria coordenação dos dois lados com complexidade muito maior [09:25] Diego. O contrato inclui documentar o comportamento de forma destacada no portal de desenvolvedores [09:26] Marcos.

## Direcionadores da Decisão

- Duplicatas de entrega são inevitáveis quando há retry e timeout.
- Não introduzir protocolo de coordenação com o cliente nesta fase.
- Dar ao cliente um identificador confiável para deduplicação.
- Seguir o padrão de mercado de provedores de webhook.

## Alternativas Consideradas

1. **At-least-once com event_id para dedup no cliente**: todo evento carrega um UUID no header X-Event-Id; o cliente descarta duplicatas pelo event_id.
2. **Exactly-once com protocolo coordenado**: ack, idempotência e reconciliação acordadas entre plataforma e cada cliente.
3. **At-most-once**: uma tentativa por evento, sem retentativa.

## Decisão

Opção 1. A entrega é at-least-once. Cada evento recebe um event_id (UUID) gerado no momento em que entra na outbox, enviado no header X-Event-Id [09:25] Diego. O identificador é único por evento: se o cliente recebe duas vezes, descarta a duplicata do lado dele [09:25] Diego. A possibilidade de duplicata fica documentada de forma destacada no portal de desenvolvedores [09:26] Marcos.

## Prós e Contras das Opções

**At-least-once com X-Event-Id**

- Prós: modelo simples e conhecido de mercado; sem protocolo extra entre as partes; a dedup pelo event_id resolve a quase totalidade dos casos [09:25] Diego.
- Contras: transfere ao cliente a responsabilidade de tratar duplicatas [09:25] Sofia; cliente mal implementado pode processar o mesmo evento duas vezes.

**Exactly-once**

- Prós: elimina a dedup do lado do cliente.
- Contras: exige coordenação dos dois lados e fica muito mais complexo [09:25] Diego; rejeitado.

**At-most-once**

- Prós: sem duplicatas e sem retry.
- Contras: perde eventos em qualquer falha transitória; incompatível com a política de retry decidida (ADR-002); rejeitado.

## Consequências

Positivas: o modelo é o mesmo de provedores estabelecidos, o que reduz a curva de adoção dos clientes; o event_id também serve de chave de correlação em suporte e observabilidade; a garantia é explícita e documentada.

Negativas: todo cliente precisa implementar dedup por event_id, requisito que passa a fazer parte do contrato de integração e da documentação pública; sem dedup do lado do cliente, efeitos colaterais duplicados são problema do integrador.

O trade-off assumido: simplicidade e robustez da plataforma contra um esforço extra conhecido e documentado do integrador.

## Referências

- `src/modules/orders/order.service.ts` (transação que origina o evento)
- `prisma/schema.prisma` (modelagem da outbox com o event_id)
- `TRANSCRICAO.md` [09:24]-[09:26] (discussão e decisão)
