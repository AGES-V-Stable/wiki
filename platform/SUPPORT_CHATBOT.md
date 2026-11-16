# Assistente Virtual de Suporte (Chatbot IA)

Este documento descreve a arquitetura, o escopo funcional, os limites operacionais, os guardrails de segurança e usabilidade (Heurísticas de Nielsen) e a estratégia de resiliência com e sem API externa do **Assistente Virtual de Suporte da V-Stable**.

---

## 1. Visão Geral e Arquitetura (`POST /v1/support/chat`)

O assistente fornece suporte contextualizado e somente leitura para usuários autenticados da plataforma V-Stable, acessível por um widget flutuante no frontend e atendido por um endpoint dedicado no backend.

```text
SupportChatWidget (React)
  └─> sendSupportChatMessage (timeout 20s / AbortController)
        └─> POST /v1/support/chat (Bearer JWT)
              └─> SupportChatController
                    └─> SupportChatService
                          ├─ Rate limit deslizante (15 req/min -> HTTP 423 Locked)
                          ├─ Contexto seguro Zero-PII (buildSafeUserContextSummary)
                          └─ System prompt de domínio (buildSystemPrompt)
                                └─> SupportChatGateway (SupportChatClient)
                                      ├─ 1. Gemini (AI_SUPPORT_PROVIDER=gemini + fallback de modelos)
                                      ├─ 2. DeepSeek (AI_SUPPORT_PROVIDER=deepseek | auto com sk-)
                                      └─ 3. Assistente Local (vstable-assistant)
```

### Componentes Principais

| Camada | Arquivo / Símbolo | Responsabilidade |
|---|---|---|
| **Frontend (UI)** | `SupportChatWidget` (`src/features/support/components/SupportChatWidget.tsx`) | Widget flutuante acessível (`Esc` fecha o painel), envio via formulário ou *chips* de perguntas frequentes, contagem preventiva de rate limit, timer regressivo ao lado do botão `Enviar`, botão `Limpar` com aborto de requisição pendente e renderização segura sem `dangerouslySetInnerHTML`. |
| **Frontend (API)** | `sendSupportChatMessage` (`src/features/support/api/supportChat.ts`) | Chamada `POST /v1/support/chat` com timeout de `20.000 ms` via `AbortController`, mapeamento de `HTTP 423` para `SupportChatRateLimitError` com extração dos segundos restantes (`parseRetryAfterSeconds`) e tratamento de `HTTP 400`. |
| **Controller** | `SupportChatController` (`POST /v1/support/chat`) | Exige `Authentication` válida (retorna `401 Unauthorized` quando ausente), valida `@Valid @RequestBody SupportChatRequestDTO` e delega ao serviço. |
| **Service** | `SupportChatService` | Aplica janela deslizante de rate limit por usuário (`15` req/min), gera o resumo seguro de sessão (`buildSafeUserContextSummary`), injeta as regras de negócio e segurança (`buildSystemPrompt`) e retorna `SupportChatResponseDTO` com `reply`, `provider`, `suggestedQuestions` e `timestamp`. |
| **Gateway** | `SupportChatGateway` | Interface de desacoplamento com o registro `SupportChatGatewayResult(String reply, String provider)`. |
| **Client & Config** | `SupportChatClient` e `SupportChatProperties` (`ai.support.*`) | Executa a cadeia de provedores (Gemini -> DeepSeek -> Assistente Local Determinístico `vstable-assistant`), normaliza alternância de turnos para o Gemini e aplica temperatura/teto de tokens conservadores. |

### Contrato da API (`POST /v1/support/chat`)

- **Request (`SupportChatRequestDTO`):**
  - `message` (`String`, obrigatório, `@NotBlank`, `@Size(max = 500)`).
  - `history` (`List<SupportChatMessageDTO>`, opcional, `@Valid`, `@Size(max = 10)`), onde cada item exige `role` igual a `"user"` ou `"assistant"` (`@Pattern(regexp = "user|assistant")`) e `content` não vazio até `4000` caracteres (`@Size(max = 4000)`).
- **Response (`SupportChatResponseDTO`):**
  - `reply`: texto em Markdown simples retornado pelo provedor ativo.
  - `provider`: identificador do provedor que respondeu (`"gemini"`, `"deepseek"` ou `"vstable-assistant"`).
  - `suggestedQuestions`: lista padrão de perguntas frequentes (`DEFAULT_SUGGESTIONS`).
  - `timestamp`: instante da resposta (`OffsetDateTime`).
- **Códigos HTTP:** `200 OK`, `400 Bad Request` (`VALIDATION_ERROR`), `401 Unauthorized` e `423 Locked` (`ACCOUNT_LOCKED`, por excesso de requisições no rate limit).

---

## 2. O que o Chatbot é Capaz de Fazer (Escopo Funcional)

O chatbot orienta usuários e administradores sobre os seguintes fluxos reais da plataforma V-Stable:

1. **Arquitetura multi-provedor e orquestração da melhor cotação:**
   - Explica a integração institucional entre **Avenia** (*on-ramp* via PIX em BRL, emissão/trânsito em stablecoins e liquidação *on-chain*/SWIFT) e **BlindPay** (*off-ramp* bancário em moeda local via parceiros como JPMorgan, Citi e Bitso).
   - Detalha o funcionamento do `QuoteOrchestratorService`, que consulta provedores elegíveis em paralelo (com timeout configurável via `quotes.provider-timeout`, padrão `10s`) para entregar a **Melhor Cotação** (*Best Quote*), exibindo taxas de rede, *spread* e IOF com trava de preço.

2. **Países, moedas e trilhos bancários suportados:**
   - **Estados Unidos (`USD`):** trilhos `ACH` (1 a 2 dias úteis), `Wire` (mesmo dia), `RTP` (*Real-Time Payments* em minutos) e `FedNow`.
   - **União Europeia / Zona do Euro (`EUR`):** `SEPA` e `SEPA Instant`.
   - **Reino Unido (`GBP`):** `Faster Payments` e `BACS`.
   - **México (`MXN`):** trilho local `SPEI` (instantâneo via integração Bitso, com chave CLABE).
   - **Argentina (`ARS`):** transferências locais via `CVU`, `CBU` ou `ALIAS` (via Bitso).
   - **Brasil (`BRL`):** depósitos e retiradas via `PIX`.
   - **Grade Global SWIFT (16 moedas homologadas em `BeneficiaryValidator.SWIFT_CURRENCIES`):** `USD`, `EUR`, `GBP`, `CAD`, `AUD`, `CHF`, `JPY`, `CNY`, `HKD`, `SGD`, `NZD`, `THB`, `PHP`, `INR`, `KRW` e `IDR`.

3. **Regra de bloqueio do peso colombiano (`COP`) e alternativas recomendadas:**
   - Esclarece que transferências bancárias diretas em `COP` estão indisponíveis porque: (a) o trilho `ACH COP` (Bitso/BlindPay) encontra-se em fase final de homologação técnica e regulatória; (b) `COP` não faz parte da grade de 16 moedas SWIFT validada em `BeneficiaryValidator`; e (c) políticas preventivas de PLD-FT/GAFI bloqueiam corredores sem liquidação automatizada homologada.
   - Orienta as duas alternativas disponíveis:
     - **Opção 1 — `Wallet Cripto` (recomendada):** cadastrar o beneficiário na modalidade de carteira Web3 em redes suportadas para receber `USDC` ou `USDT`.
     - **Opção 2 — Conta Internacional em `USD` ou `EUR`:** cadastrar o beneficiário recebendo em `USD` (`ACH`/`Wire`/`SWIFT`) ou `EUR` (`SEPA`/`SWIFT`).

4. **Redes blockchain e stablecoins:**
   - **Stablecoins:** `USDC`, `USDT`, `BRLA` e `EURC`.
   - **Redes blockchain:** `Polygon`, `Celo`, `Ethereum`, `Base`, `Tron`, `Gnosis` e `Moonbeam` (observando que o enum `BlockchainNetwork` no cadastro de carteiras cobre `ethereum`, `polygon`, `celo`, `gnosis`, `moonbeam` e `tron`, enquanto a arquitetura Avenia contempla também `Base`).

5. **Autenticação em Dois Fatores (`2FA / TOTP`):**
   - Explica a obrigatoriedade do 2FA no login, na ativação de conta e no Passo 3 (*Autenticação*) de transferências.
   - Orienta a troca de aparelho / novo QR Code em **Configurações > Segurança da Conta**, informando a exigência do código TOTP atual de 6 dígitos (que revoga imediatamente o segredo anterior) e a necessidade de remover o registro antigo no aplicativo autenticador.
   - Informa o bloqueio temporário de **15 minutos** após **5 tentativas incorretas** (`TwoFactorAuthService`: `MAX_FAILED_ATTEMPTS = 5`, `LOCKOUT_SECONDS = 15 * 60`) e a proteção **anti-replay atômica em PostgreSQL** na tabela `totp_used_steps` (`TotpReplayStore`), que garante uso único por janela de **90 segundos** (passo atual de 30s com tolerância de `±1` passo em `TotpService`) compartilhado entre instâncias Lambda.

6. **Onboarding, Termos de Serviço (`ToS`) da BlindPay e KYC Individual / *Liveness*:**
   - Orienta as etapas de onboarding (cadastro da empresa, representante legal, ativação de 2FA, aceite dos Termos de Serviço `ToS` da BlindPay e envio de documento de identidade + prova de vida *Liveness*), esclarecendo que a situação do cadastro depende do KYC individual e não há exigência local de KYB ou AML da empresa.
   - Documenta o limite de upload de **3 MB por arquivo** (`spring.servlet.multipart.max-file-size: 3MB` / `BlindPayIndividualKycService.MAX_FILE_BYTES = 3 * 1024 * 1024`) e a possibilidade de **reiniciar/reabrir o KYC** individual quando necessário (`POST /v1/blindpay/kyc/restart`).

7. **Gestão de membros da empresa, papéis, convites e revogação:**
   - Explica os três papéis de `CompanyRole`: `OWNER` (gestão completa de membros, convites, alteração de papéis, revogação e operações financeiras), `OPERATOR` (operação de beneficiários, cotações e transferências) e `VIEWER` (somente visualização/leitura).
   - Orienta o fluxo de convites por link criptografado com aceite obrigatório dos Termos de Uso e Política de Privacidade vigentes (`TERMS_VERSION`), alteração de papel e revogação imediata de vínculo (`MembershipStatus.REMOVED`).

8. **Saldo empresarial, créditos simulados, dashboard, comprovante em PDF e conciliação:**
   - Explica o uso do saldo empresarial em BRL (`companies.available_balance_brl`) para liquidação de transferências e o suporte, em ambientes habilitados (`balance.simulated-deposits.enabled=true`), a créditos simulados protegidos por `Idempotency-Key` (8 a 128 caracteres) com emissão de recibo (`mock_topup_receipts`).
   - Explica as cotações de referência exibidas no dashboard (`BRL -> USDC` e `BRL -> USDT`, rota `PIX -> POLYGON`, usando `basePrice` sem tarifas, com cache de **2 horas** de TTL em `exchange_rate_snapshots`) e o cálculo de economia estimada (`SavingsEstimator`) frente ao **spread bancário de referência de 4%** (`dashboard.savings.reference-spread-percentage: 4.00`).
   - Orienta sobre a emissão de **comprovante em PDF** (`TransferReceiptPdfGenerator`) e sobre a rotina de **conciliação automática em segundo plano** (`AveniaTransactionReconciler` consultando tickets pendentes e `TransferExecutionReconciler` retomando execuções BlindPay paradas com idempotência).

---

## 3. O que o Chatbot NÃO é Capaz de Fazer (Limites Explícitos)

Por projeto arquitetural e segurança operacional, o chatbot possui fronteiras estritas:

- **Exclusivamente orientativo e somente leitura:** não cria, aprova, cancela ou executa transferências; não gera cotações operacionais em tempo real; não altera cadastros ou papéis; não gera QR Codes de 2FA; e não cria, regenera ou revoga convites ou vínculos empresariais.
- **Sem acesso a dados financeiros ou sensíveis:** não consulta nem exibe saldos (`available_balance_brl`), extratos, histórico de transações, CPF, CNPJ, e-mails, senhas, hashes (`password_hash`/`password_salt`), segredos TOTP (`two_factor_secret`) ou tokens de convite (`token_ciphertext`/`token_hash`).
- **Sem visibilidade entre empresas ou vínculos encerrados:** não acessa dados de outras empresas e não expõe dados empresariais para usuários sem vínculo ou com vínculo `REMOVED` (mesmo quando `companyId` legado ainda exista na entidade do usuário).
- **Recusa de uso indevido:** recusa pedidos fora do domínio V-Stable, solicitações de execução de código ou comandos, pedidos de dados de terceiros e tentativas de *prompt injection* para ignorar instruções do sistema.

---

## 4. Guardrails de Segurança, Privacidade e Usabilidade (Heurísticas de Nielsen)

### 4.1 Isolamento Multi-Tenant e Zero-PII (`buildSafeUserContextSummary`)

Antes de chamar qualquer provedor externo ou o assistente local, `SupportChatService.buildSafeUserContextSummary` constrói um resumo mínimo de sessão com **Zero-PII**:

- **Para `UserEntity`:**
  - Resolve a empresa exclusivamente via `activeMembershipCompanyId(user)`, que exige `user.getMembership() != null` e `membership.getStatus() == MembershipStatus.ACTIVE`.
  - Vínculos `REMOVED`, ausência de vínculo ou o campo legado `user.getCompanyId()` são ignorados e resultam na não consulta ao `CompanyRepository`, enviando apenas: `**Status atual da sua conta:** 2FA = Ativo|Pendente de ativação | Empresa ainda não vinculada.`
  - Com vínculo `ACTIVE` e empresa localizada, envia somente o nome fantasia (`tradeName`) ou razão social (`legalName`), o papel (`Papel = OWNER|OPERATOR|VIEWER`) e o booleano do 2FA (`2FA = Ativo|Pendente de ativação`).
- **Para `AdministratorEntity`:**
  - Envia apenas `**Status da sua sessão (Administrador):** 2FA = Ativo|Inativo.`
- **Dados nunca enviados ao prompt ou provedores de IA:** nome completo, e-mail, CPF, CNPJ, telefone, endereço, UUIDs internos (`userId`, `companyId`, `membershipId`), saldo `available_balance_brl`, hashes/salts de senha, segredos TOTP e tokens de convite.

### 4.2 Validação de Entrada e Controles de Geração

| Camada | Limite / Regra Aplicada |
|---|---|
| **DTO Backend (`SupportChatRequestDTO`)** | `message` obrigatório (`@NotBlank`) com no máximo `500` caracteres (`@Size(max = 500)`); `history` com no máximo `10` itens (`@Size(max = 10)`). |
| **DTO de Mensagem (`SupportChatMessageDTO`)** | `role` restrito a `user` ou `assistant` (`@Pattern(regexp = "user|assistant")`); `content` obrigatório com até `4000` caracteres (`@Size(max = 4000)`). |
| **Widget Frontend (`SupportChatWidget`)** | `<input maxLength={500} />`; histórico enviado limitado às últimas `6` mensagens (excluindo a mensagem inicial de boas-vindas), cada uma truncada em `800` caracteres. |
| **Parâmetros LLM (`SupportChatClient`)** | Gemini: `temperature = 0.2`, `maxOutputTokens = 500`; DeepSeek: `temperature = 0.25`, `max_tokens = 600`; instrução de respostas concisas (100 a 180 palavras). |

### 4.3 Sanitização contra XSS no Frontend

O helper `renderFormattedText` em `SupportChatWidget.tsx` **não usa `dangerouslySetInnerHTML`** nem injeta HTML bruto no DOM. Ele divide cada linha apenas pelos marcadores `**negrito**` (`<strong>`) e `` `código` `` (`<code>`), deixando que o React escape qualquer tag ou script (como `<script>...</script>`) como texto literal.

### 4.4 Rate Limit no Backend e Heurísticas de Nielsen no Frontend

- **Rate limit no backend (`SupportChatService.enforceRateLimit`):**
  - Mantém janela deslizante em memória (`ConcurrentHashMap<String, Deque<Instant>>`) com teto de **`15` requisições por minuto** (`MAX_REQUESTS_PER_MINUTE = 15`, `RATE_LIMIT_WINDOW_SECONDS = 60L`) por chave normalizada de usuário (`user:<email>`, `admin:<email>` ou `principal:<string>`).
  - Quando o limite é atingido, calcula o tempo restante até a expiração do timestamp mais antigo na janela e lança `LockedException` com a mensagem `"Limite de mensagens atingido. Aguarde %d segundos antes de enviar nova pergunta."`, traduzida pelo `GlobalExceptionHandler` para **`HTTP 423 Locked`** (`code: "ACCOUNT_LOCKED"`).
- **Aplicação das Heurísticas de Nielsen no frontend (`SupportChatWidget` e `supportChat.ts`):**
  1. **Heurística #1 — Visibilidade do status do sistema:**
     - Exibe um **timer regressivo** (`data-testid="support-chat-cooldown-timer"`, `role="timer"`, `aria-live="polite"`) mostrando `{cooldownSeconds}s` diretamente ao lado do botão `Enviar` durante todo o período de espera.
     - Exibe também o indicador de processamento `"Analisando sua dúvida..."` (`role="status"`) durante chamadas ativas e o status `"Suporte inteligente • Online"` no cabeçalho.
  2. **Heurística #5 — Prevenção de erros:**
     - O widget mantém contagem preventiva local (`sentTimestampsRef`) com os mesmos `15` envios em janela deslizante de `60.000 ms` (`SUPPORT_CHAT_MAX_REQUESTS_PER_MINUTE = 15`, `SUPPORT_CHAT_RATE_LIMIT_WINDOW_MS = 60_000`), ativando o cooldown preventivamente antes de disparar requisição excedente ao servidor.
     - Enquanto `cooldownSeconds > 0`, desabilita o botão `Enviar` (`disabled={loading || cooldownSeconds > 0 || !input.trim()}`) e os botões de perguntas frequentes (`disabled={cooldownSeconds > 0}`), impedindo novos envios até o término da contagem.
  3. **Heurística #9 — Ajudar usuários a reconhecer, diagnosticar e recuperar-se de erros:**
     - Quando uma resposta `HTTP 423` chega do backend, `parseRetryAfterSeconds` extrai a quantidade de segundos da mensagem (ou usa `error.retryAfterSeconds` / fallback de `60s`) e lança `SupportChatRateLimitError`.
     - O widget exibe o alerta amigável (`role="alert"`), decrementa `cooldownSeconds` a cada segundo e **remove automaticamente o alerta de rate limit e libera os envios** assim que o contador chega a `0` (`visibleError = error && !(error.isRateLimit && cooldownSeconds === 0) ? error.message : null`).
- **Timeout e cancelamento:** `sendSupportChatMessage` aborta requisições que excedam `20` segundos (`SUPPORT_CHAT_TIMEOUT_MS = 20_000`) via `AbortController`, enquanto `SupportChatWidget` aborta chamadas pendentes ao clicar em `Limpar` ou desmontar o componente, descartando respostas tardias.

---

## 5. Dependência de API Externa e Funcionamento Offline / Sem API

**Funciona sem API externa? Sim.**

O `SupportChatClient` implementa uma cadeia de resiliência em 3 níveis que impede indisponibilidade total do assistente caso não haja chave configurada ou ocorram falhas nos provedores externos:

| Nível | Provedor (`response.provider`) | Condição de Acionamento | Endpoint / Mecanismo |
|---|---|---|---|
| **1. Primário** | `gemini` | `AI_SUPPORT_API_KEY` preenchida e `AI_SUPPORT_PROVIDER` igual a `gemini` (padrão) ou `auto` (quando a chave não inicia com `sk-`). | `POST {geminiBaseUrl}/v1beta/models/{model}:generateContent` com header `x-goog-api-key`. Tenta o modelo configurado em `AI_SUPPORT_GEMINI_MODEL` (padrão `gemini-flash-lite-latest`) e aplica fallback sequencial entre `gemini-flash-lite-latest`, `gemini-flash-latest`, `gemini-3.5-flash-lite` e `gemini-3.5-flash` em respostas `404`, `429`, `503`, respostas vazias ou falhas de rede (`RestClientException`). |
| **2. Alternativo** | `deepseek` | `AI_SUPPORT_API_KEY` preenchida e `AI_SUPPORT_PROVIDER=deepseek` ou `AI_SUPPORT_PROVIDER=auto` com chave iniciando em `sk-`. | `POST {deepseekBaseUrl}/v1/chat/completions` com `Authorization: Bearer {apiKey}` e modelo `AI_SUPPORT_DEEPSEEK_MODEL` (padrão `deepseek-chat`). |
| **3. Offline / Determinístico** | `vstable-assistant` | `AI_SUPPORT_API_KEY` vazia/ausente, `AI_SUPPORT_PROVIDER=local` ou falha/esgotamento de todos os modelos dos provedores externos. | `SupportChatClient.generateBuiltInKnowledgeReply(userMessage, userContextSummary)`: classifica palavras-chave normalizadas sem acentuação e retorna respostas determinísticas embarcadas sobre Avenia/BlindPay, países/moedas/trilhos, bloqueio `COP`, redes/stablecoins, 2FA/anti-replay, KYC/ToS/reabertura, cotações/saldo/PDF/conciliação, membros/convites e dashboard/economia estimada, anexando o mesmo contexto seguro da sessão. |

### Variáveis de Ambiente (`SupportChatProperties` / `application.yaml`)

| Variável | Propriedade Spring | Padrão |
|---|---|---|
| `AI_SUPPORT_PROVIDER` | `ai.support.provider` | `gemini` (`gemini`, `deepseek`, `auto` ou `local`) |
| `AI_SUPPORT_API_KEY` | `ai.support.api-key` | `""` (vazio ativa `vstable-assistant` imediatamente) |
| `AI_SUPPORT_GEMINI_MODEL` | `ai.support.gemini-model` | `gemini-flash-lite-latest` |
| `AI_SUPPORT_GEMINI_BASE_URL` | `ai.support.gemini-base-url` | `https://generativelanguage.googleapis.com` |
| `AI_SUPPORT_DEEPSEEK_MODEL` | `ai.support.deepseek-model` | `deepseek-chat` |
| `AI_SUPPORT_DEEPSEEK_BASE_URL` | `ai.support.deepseek-base-url` | `https://api.deepseek.com` |
