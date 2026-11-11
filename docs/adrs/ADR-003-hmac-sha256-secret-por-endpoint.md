# ADR-003: Autenticação por HMAC-SHA256 com secret por endpoint e rotação

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:22] Sofia)
- **Decisores:** Sofia, Diego, Larissa, Bruno

## Contexto

Os eventos carregam dados de pedidos para um endpoint fora da infraestrutura da empresa. O cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou uma secret em log da própria aplicação ([09:22] Diego).

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256**, enviando a assinatura no header `X-Signature` ([09:20] Sofia).
- Usar **uma secret única por endpoint de webhook**, nunca uma secret global ([09:21] Sofia). A secret é gerada pela plataforma e devolvida na criação ([09:31] Marcos).
- Permitir **rotação pela API**: ao rotacionar, a secret antiga continua válida por **24 horas** em paralelo e depois deixa de valer ([09:21] Sofia).
- A URL do webhook deve ser **https**; `http` é recusado por validação no schema Zod. Não é decisão arquitetural, é validação ([09:23] Sofia).
- O header `X-Timestamp` (momento do envio) permite ao cliente detectar replay attack, se quiser ([09:44] Diego).

## Alternativas Consideradas

1. **Secret global da plataforma.** Descartada: o vazamento de uma secret comprometeria todos os clientes ([09:21] Sofia).
2. **Secret por endpoint sem rotação.** Descartada: sem rotação, um vazamento não tem remediação, e o histórico mostra que vazamentos acontecem ([09:22] Diego).
3. **Sem assinatura, confiando apenas em TLS.** Descartada: o requisito é que o cliente consiga provar a origem e a integridade do payload ([09:19] Sofia).

## Consequências

**Positivas**
- Vazamento isolado a um único endpoint.
- Padrão de mercado, com bibliotecas disponíveis em qualquer stack de cliente ([09:20] Sofia).
- A rotação com grace period permite migração sem downtime do lado do cliente.

**Negativas / trade-offs**
- Durante a janela de 24 horas há duas secrets válidas, o que aumenta a complexidade do envio e da verificação.
- O HMAC exige a secret em claro para assinar, então ela precisa ser armazenável pela plataforma. Como protegê-la em repouso **não foi discutido** e fica para a revisão de segurança ([09:46] Sofia reserva dois dias úteis para revisar HMAC e geração de secret).
- Como o envio assina durante o grace period (uma ou duas assinaturas) **não foi definido** na reunião; ver Questões em Aberto do [RFC](../RFC.md).
