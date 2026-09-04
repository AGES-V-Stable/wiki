Documentação do Banco de Dados: V-Stable
Visão Geral da Arquitetura
O banco de dados da V-Stable foi modelado em PostgreSQL.

Padrões Adotados:

Identificadores: Utilização de UUID v4 nativo para todas as chaves primárias (PK), garantindo segurança e impossibilitando a enumeração de registros em rotas públicas.

Padronização via ENUMs: Controle rígido de estados (Status de Transação, Compliance, Métodos de Transferência e Redes Blockchain) diretamente no motor do banco.

Herança 1-para-1 (Table-per-Type): Separação elegante do fluxo de transações. Os dados financeiros comuns ficam na tabela transacoes_base, enquanto campos específicos de envio (Importação) e recebimento (Exportação/Invoice) ficam em tabelas filhas, garantindo a normalização e eliminando colunas vazias.

Dicionário de Tabelas
1. Núcleo e Identidade (Core)
empresas: Tabela central do sistema. Armazena os dados cadastrais da PME, endereço, informações de compliance (propósito, receita) e o saldo disponível na plataforma.

Destaque: Possui controle de aprovação de compliance granularizado por moeda (kyc_usd_aprovado e kyc_cop_aprovado).
Campos necessárarios devido a diferenças de compliance para realizar transações com Dolar e Peso Colombiano, entre outras.

usuarios: Representantes legais que operam o painel da PME.

Destaque: Centraliza a segurança da conta (credenciais, status de atividade e segredo_2fa para validação TOTP no backend).

administradores: Usuários internos da V-Stable (staff). Tabela isolada das PMEs, com níveis de acesso definidos (SUPER_ADMIN, ANALISTA_COMPLIANCE, SUPORTE) e exigências rígidas de segurança (troca de senha obrigatória e 2FA).

2. Onboarding e Compliance
progresso_cadastros: Tabela de Draft (Rascunho) utilizada durante o onboarding. Permite que o usuário avance nas etapas do cadastro salvando o progresso sem ferir as restrições NOT NULL das tabelas oficiais.

Destaque: Utiliza o tipo JSONB (dados_temporarios) para armazenar os payloads parciais. Ao finalizar, o backend converte o JSON e consolida nas tabelas empresas e usuarios.

documentos_compliance: Registro dos arquivos (Contrato Social, Comprovantes de Endereço, CNH) enviados para auditoria. Armazena a URL do bucket S3 e o status de aprovação de cada arquivo.

3. Diretório Financeiro
beneficiarios: Agenda de contatos unificada por PME. Suporta múltiplos destinos financeiros em um único registro.

Destaque: Registra identificadores da API parceira (avenia_id para contas fiduciárias e avenia_wallet_id para carteiras). Suporta contas bancárias tradicionais, PIX e redes Blockchain (Ethereum, Polygon, Tron, etc.), exigindo o preenchimento apenas dos dados correspondentes ao tipo_recebimento.

4. Motor de Transações (Herança)
transacoes_base: Concentra o núcleo monetário de qualquer movimentação.

Destaque: Armazena a conversão exata da operação (moeda_estrangeira, valor_estrangeiro, valor_liquidacao_brl, taxa_cambio, percentual_spread_efetivo). Guarda o avenia_ticket_id para conciliação via Webhooks e o hash on-chain da blockchain.

transacoes_importacao: Extensão da transação para fluxos de Saída (Pagamento de fornecedores).

Destaque: A chave primária é uma FK (transacao_id). Exige o vínculo direto com a tabela de beneficiarios e a definição do método de pagamento da PME.

transacoes_exportacao: Extensão da transação para fluxos de Entrada (Geração de Invoices/Cobranças).

Destaque: Não utiliza a tabela de beneficiários. Armazena o link de pagamento único (codigo_cobranca_externa), os dados do cliente internacional e a data de vencimento da fatura.

Regras de Negócio e Integridade (CHECK Constraints)
Para blindar o banco de dados contra inconsistências lógicas vindas do backend, foram implementadas as seguintes restrições nativas:

Integridade Direcional de Transações (chk_direcao_transacao)

Se direcao_operacao = IMPORTACAO: A transação deve possuir um beneficiário de destino e não pode conter dados de geração de fatura externa.

Se direcao_operacao = EXPORTACAO: A transação não pode possuir beneficiário e deve conter obrigatoriamente um código de cobrança (invoice), nome do pagador e vencimento.

Consistência de Destinos Financeiros (chk_dados_recebimento)

Valida a completude dos dados do beneficiário com base no tipo escolhido.

Se WALLET_CRYPTO: Exige endereço e rede blockchain.

Se CHAVE_PIX: Exige a chave preenchida.

Se CONTA_BANCARIA: Exige o preenchimento de todo o formulário SWIFT/Local (documento, titular, banco, agência, conta e tipo).
