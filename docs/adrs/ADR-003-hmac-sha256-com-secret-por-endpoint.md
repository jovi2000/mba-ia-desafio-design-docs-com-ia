# ADR-003: HMAC-SHA256 com secret por endpoint e rotação com grace period

**Status:** Aceito
**Data:** 2026-11-22
**ADRs relacionados:** ADR-001, ADR-004

## Contexto

Os eventos de webhook carregam dados de pedidos e são entregues em endpoints fora da infraestrutura da plataforma. O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que ninguém adulterou o payload no caminho [09:19] Sofia. Sem um mecanismo de autenticação de aplicação, qualquer um que descubra a URL do endpoint pode forjar eventos.

A discussão estabeleceu o mecanismo: assinar o payload com uma secret compartilhada entre plataforma e cliente, enviando a assinatura no header X-Signature [09:20] Sofia, com HMAC-SHA256, padrão de mercado com biblioteca disponível em qualquer stack [09:20] Sofia. Ficou explícito que a secret é por endpoint, não global, para conter o raio de um vazamento [09:21] Sofia, e que precisa ser rotacionável: quando o cliente pede nova secret, a antiga permanece válida por 24 horas em paralelo, para dar tempo de migrar os sistemas dele, e depois é desativada [09:21] Sofia. Houve precedente real de cliente que vazou secret em log de aplicação [09:22] Diego.

## Direcionadores da Decisão

- Autenticar a origem do evento para o cliente que recebe.
- Garantir a integridade do payload entregue.
- Conter o raio de explosão de um vazamento de credencial.
- Permitir rotação de secret sem quebrar a integração do cliente.

## Alternativas Consideradas

1. **HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h**: cada cadastro de webhook tem secret própria; a rotação mantém a antiga válida por 24 horas.
2. **Secret global da plataforma**: uma única secret compartilhada com todos os clientes.
3. **Nenhuma assinatura de aplicação**: confiar apenas no TLS e na obscuridade da URL.

## Decisão

Opção 1. Todo envio é assinado com HMAC-SHA256 sobre o corpo do request, com a assinatura no header X-Signature. Cada endpoint de webhook tem uma secret única, gerada pela plataforma na criação do cadastro e devolvida ao cliente nesse momento [09:31] Marcos. A tabela de configuração armazena url, secret, customer_id e estado ativo [09:21] Bruno. A rotação é solicitada pelo cliente via endpoint próprio; a secret antiga permanece válida por 24 horas em paralelo e depois é desativada [09:21] Sofia.

## Prós e Contras das Opções

**HMAC-SHA256 por endpoint com rotação**

- Prós: vazamento de uma secret afeta um único cadastro; rotação sem indisponibilidade de integração; verificação simples do lado do cliente com biblioteca padrão.
- Contras: o cliente precisa implementar a verificação; durante o grace period coexistem duas secrets válidas, ampliando a superfície momentânea de aceitação.

**Secret global**

- Prós: operação mais simples, um único segredo para gerenciar.
- Contras: se vaza uma, vaza tudo [09:21] Sofia; vazamento único já ocorreu no histórico de clientes [09:22] Diego; rejeitada.

**Nenhuma assinatura**

- Prós: zero gestão de credenciais.
- Contras: não autentica a origem nem protege contra adulteração por quem descobre a URL; não atende ao requisito de segurança [09:19] Sofia; rejeitada.

## Consequências

Positivas: cada cliente verifica autenticidade e integridade com um mecanismo que é padrão de mercado, com biblioteca disponível em qualquer stack [09:20] Sofia; o comprometimento de credencial tem alcance de um endpoint; a rotação é self service.

Negativas: o grace period de 24 horas mantém a secret antiga aceita durante a migração, uma janela de risco assumida em troca de não quebrar integrações em produção; a plataforma passa a guardar segredos por cliente, o que exige disciplina de armazenamento e de log (o logger Pino já aplica redact a campos sensíveis, padrão a manter); geração e validação de HMAC entram na revisão de segurança obrigatória antes do deploy, com dois dias úteis reservados [09:46] Sofia.

O trade-off assumido: gestão de segredos por cliente e janela de dupla validade em troca de contenção de vazamento e rotação sem downtime.

## Referências

- `src/shared/logger/index.ts` (redact já configurado, a manter para segredos)
- `prisma/schema.prisma` (tabela de configuração de webhook a modelar)
- `TRANSCRICAO.md` [09:19]-[09:22] (discussão e decisão)
