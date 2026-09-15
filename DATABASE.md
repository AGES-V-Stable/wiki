# Banco de Dados

O banco de dados da V-Stable é modelado em **PostgreSQL**, com o schema aplicado via **Flyway** (`src/main/resources/db/migration/V1__init.sql`).

## Diagramas

<div align="center">

  <img width="2071" height="1441" alt="diagrama-vstable-v8" src="https://github.com/user-attachments/assets/1cd6d50a-2845-43c7-b5e8-68822673f5b0" />

  <br/>
  <em>Modelo Lógico de Dados</em>
</div>

<div align="center">
  
  <img width="1084" height="666" alt="ER Diagram0" src="https://github.com/user-attachments/assets/5a0146e0-599c-4407-b522-2d523a6f38ed" />
<br/>
  <em>Modelo Conceitual</em>
</div>

## Padrões Adotados

- **Identificadores:** `UUID v4` nativo em todas as chaves primárias (PK), garantindo segurança e impossibilitando a enumeração de registros em rotas públicas.
- **Padronização via ENUMs:** controle rígido de estados (Status de Transação, Compliance, Métodos de Transferência e Redes Blockchain) diretamente no motor do banco.
- **Herança 1-para-1 (Table-per-Type):** os dados financeiros comuns ficam em `base_transactions`, enquanto campos específicos de envio (Importação) e recebimento (Exportação/Invoice) ficam em tabelas filhas, mantendo a normalização e evitando colunas vazias.
- **Nomenclatura em inglês:** tabelas, colunas e tipos ENUM são nomeados em inglês. Os valores dos ENUMs e os campos das APIs continuam em português.

## Dicionário de Tabelas

### 1. Núcleo e Identidade

> O cadastro é feito em uma única chamada atômica (`POST /v1/cadastros/onboarding`) — as entidades de Empresa e Usuário nascem simultaneamente nessa chamada, junto com um registro pendente de verificação de KYC.

**`companies`** — tabela central do sistema. Armazena os dados cadastrais da PME, endereço, informações de compliance (propósito, receita) e o saldo disponível na plataforma. Possui controle de aprovação de compliance granularizado por moeda (`kyc_usd_approved` e `kyc_cop_approved`), necessário pelas diferenças de compliance para transações em Dólar e Peso Colombiano, entre outras.

**`users`** — representantes legais que operam o painel da PME. Centraliza a segurança da conta (`password_hash`/`password_salt`, status `active` e `two_factor_secret` para validação TOTP no backend). Exige obrigatoriamente o vínculo com uma empresa (`company_id`).

**`administrators`** — usuários internos da V-Stable (staff). Tabela isolada das PMEs, com níveis de acesso definidos (`SUPER_ADMIN`, `ANALISTA_COMPLIANCE`, `SUPORTE`) e exigências rígidas de segurança (troca de senha obrigatória e 2FA).

### 2. Onboarding e Compliance

**`avenia_kyc_verifications`** — registro da verificação de reconhecimento facial (Avenia). Criada automaticamente com `status = PENDENTE` ao final do onboarding, representando o próximo passo obrigatório (KYC facial) antes da liberação completa da conta.

**`compliance_documents`** — registro dos arquivos (Contrato Social, Comprovantes de Endereço, Documento do Representante) enviados para auditoria. Armazena a URL do arquivo e o status de aprovação de cada um.

### 3. Diretório Financeiro

**`beneficiaries`** — agenda de contatos unificada por PME, suportando múltiplos destinos financeiros em um único registro. Registra identificadores da API parceira (`avenia_id` para contas fiduciárias e `avenia_wallet_id` para carteiras). Suporta contas bancárias tradicionais, PIX (`pix_key`) e redes Blockchain (Ethereum, Polygon, Tron, etc. via `blockchain_network`), conforme o `receiving_method` escolhido.

### 4. Motor de Transações (Herança Table-per-Type)

**`base_transactions`** — concentra o núcleo monetário de qualquer movimentação (entrada ou saída). Armazena a conversão exata da operação (`foreign_currency`, `foreign_amount`, `settlement_amount_brl`, `exchange_rate`, `effective_spread_percentage`), o `avenia_ticket_id` para conciliação via Webhooks e o `blockchain_transaction_hash` para rastreabilidade on-chain.

**`import_transactions`** — extensão da transação para fluxos de Saída (pagamento de fornecedores). A chave primária é a própria FK (`transaction_id`). Exige o vínculo obrigatório com `beneficiaries` e a definição do `transfer_method` (TED, PIX, BLOCKCHAIN, SALDO_EM_CONTA) usado para enviar os fundos.

**`export_transactions`** — extensão da transação para fluxos de Entrada (geração de Invoices/Cobranças Internacionais). Totalmente desacoplada de `beneficiaries` (já que o dinheiro entra). Armazena o código de cobrança único (`external_billing_code`), os dados de contato do cliente internacional e a `due_date` da fatura.

## Regras de Negócio e Integridade

**Isolamento estrutural de transações** — em vez de depender de validações frágeis em uma única tabela gigantesca, a própria divisão física em `import_transactions` (exigindo Beneficiário) e `export_transactions` (exigindo Dados da Fatura) garante a integridade direcional da operação logo na modelagem, impedindo a inserção cruzada de dados.

**Consistência de destinos financeiros (`beneficiaries`)** — regra de negócio esperada conforme o `receiving_method` escolhido:

| `receiving_method` | Exige |
|---|---|
| `WALLET_CRYPTO` | Endereço da carteira e rede blockchain |
| `CHAVE_PIX` | Chave Pix preenchida |
| `CONTA_BANCARIA` | Formulário completo SWIFT/Local (documento de identificação, titular, código do banco, agência, conta e tipo de conta) |
