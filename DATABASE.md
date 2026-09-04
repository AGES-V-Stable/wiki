<h1>🗄️ Documentação do Banco de Dados: V-Stable</h1>

<h2>🏗️ Visão Geral da Arquitetura</h2>
<p>O banco de dados da V-Stable foi modelado em <strong>PostgreSQL</strong>.</p>

<div align="center">
  <img width="100%" alt="modelagem_logica_vstable" src="https://github.com/user-attachments/assets/1e474744-2310-4457-af00-7dd8ff5db672" />
  <br/>
  <em>Modelo Lógico de Dados</em>
</div>

<h3>Padrões Adotados</h3>
<ul>
  <li><strong>Identificadores:</strong> Utilização de <code>UUID v4</code> nativo para todas as chaves primárias (PK), garantindo segurança e impossibilitando a enumeração de registros em rotas públicas.</li>
  <li><strong>Padronização via ENUMs:</strong> Controle rígido de estados (Status de Transação, Compliance, Métodos de Transferência e Redes Blockchain) diretamente no motor do banco.</li>
  <li><strong>Herança 1-para-1 (Table-per-Type):</strong> Separação elegante do fluxo de transações. Os dados financeiros comuns ficam na tabela <code>transacoes_base</code>, enquanto campos específicos de envio (Importação) e recebimento (Exportação/Invoice) ficam em tabelas filhas, garantindo a normalização e eliminando colunas vazias.</li>
</ul>

<hr>

<h2>📖 Dicionário de Tabelas</h2>

<h3>1. Núcleo e Identidade (Core)</h3>
<ul>
  <li><strong><code>empresas</code>:</strong> Tabela central do sistema. Armazena os dados cadastrais da PME, endereço, informações de compliance (propósito, receita) e o saldo disponível na plataforma.
    <blockquote><strong>💡 Destaque:</strong> Possui controle de aprovação de compliance granularizado por moeda (<code>kyc_usd_aprovado</code> e <code>kyc_cop_aprovado</code>). Campos necessários devido a diferenças de compliance para realizar transações com Dólar e Peso Colombiano, entre outras.</blockquote>
  </li>
  <li><strong><code>usuarios</code>:</strong> Representantes legais que operam o painel da PME.
    <blockquote><strong>💡 Destaque:</strong> Centraliza a segurança da conta (credenciais, status de atividade e <code>segredo_2fa</code> para validação TOTP no backend).</blockquote>
  </li>
  <li><strong><code>administradores</code>:</strong> Usuários internos da V-Stable (staff). Tabela isolada das PMEs, com níveis de acesso definidos (<code>SUPER_ADMIN</code>, <code>ANALISTA_COMPLIANCE</code>, <code>SUPORTE</code>) e exigências rígidas de segurança (troca de senha obrigatória e 2FA).</li>
</ul>

<h3>2. Onboarding e Compliance</h3>
<ul>
  <li><strong><code>progresso_cadastros</code>:</strong> Tabela de Draft (Rascunho) utilizada durante o onboarding. Permite que o usuário avance nas etapas do cadastro salvando o progresso sem ferir as restrições <code>NOT NULL</code> das tabelas oficiais.
    <blockquote><strong>💡 Destaque:</strong> Utiliza o tipo <code>JSONB</code> (<code>dados_temporarios</code>) para armazenar os payloads parciais. Ao finalizar, o backend converte o JSON e consolida nas tabelas <code>empresas</code> e <code>usuarios</code>.</blockquote>
  </li>
  <li><strong><code>documentos_compliance</code>:</strong> Registro dos arquivos (Contrato Social, Comprovantes de Endereço, CNH) enviados para auditoria. Armazena a URL do bucket S3 e o status de aprovação de cada arquivo.</li>
</ul>

<h3>3. Diretório Financeiro</h3>
<ul>
  <li><strong><code>beneficiarios</code>:</strong> Agenda de contatos unificada por PME. Suporta múltiplos destinos financeiros em um único registro.
    <blockquote><strong>💡 Destaque:</strong> Registra identificadores da API parceira (<code>avenia_id</code> para contas fiduciárias e <code>avenia_wallet_id</code> para carteiras). Suporta contas bancárias tradicionais, PIX e redes Blockchain (Ethereum, Polygon, Tron, etc.), exigindo o preenchimento apenas dos dados correspondentes ao <code>tipo_recebimento</code>.</blockquote>
  </li>
</ul>

<h3>4. Motor de Transações (Herança)</h3>
<ul>
  <li><strong><code>transacoes_base</code>:</strong> Concentra o núcleo monetário de qualquer movimentação.
    <blockquote><strong>💡 Destaque:</strong> Armazena a conversão exata da operação (<code>moeda_estrangeira</code>, <code>valor_estrangeiro</code>, <code>valor_liquidacao_brl</code>, <code>taxa_cambio</code>, <code>percentual_spread_efetivo</code>). Guarda o <code>avenia_ticket_id</code> para conciliação via Webhooks e o hash on-chain da blockchain.</blockquote>
  </li>
  <li><strong><code>transacoes_importacao</code>:</strong> Extensão da transação para fluxos de Saída (Pagamento de fornecedores).
    <blockquote><strong>💡 Destaque:</strong> A chave primária é uma FK (<code>transacao_id</code>). Exige o vínculo direto com a tabela de <code>beneficiarios</code> e a definição do método de pagamento da PME.</blockquote>
  </li>
  <li><strong><code>transacoes_exportacao</code>:</strong> Extensão da transação para fluxos de Entrada (Geração de Invoices/Cobranças).
    <blockquote><strong>💡 Destaque:</strong> Não utiliza a tabela de beneficiários. Armazena o link de pagamento único (<code>codigo_cobranca_externa</code>), os dados do cliente internacional e a data de vencimento da fatura.</blockquote>
  </li>
</ul>

<hr>

<h2>🛡️ Regras de Negócio e Integridade (<code>CHECK Constraints</code>)</h2>
<p>Para blindar o banco de dados contra inconsistências lógicas vindas do backend, foram implementadas as seguintes restrições nativas no PostgreSQL:</p>

<h3>Integridade Direcional de Transações (<code>chk_direcao_transacao</code>)</h3>
<ul>
  <li><strong>Se <code>direcao_operacao = IMPORTACAO</code>:</strong> A transação deve possuir um beneficiário de destino e não pode conter dados de geração de fatura externa.</li>
  <li><strong>Se <code>direcao_operacao = EXPORTACAO</code>:</strong> A transação não pode possuir beneficiário e deve conter obrigatoriamente um código de cobrança (invoice), nome do pagador e vencimento.</li>
</ul>

<h3>Consistência de Destinos Financeiros (<code>chk_dados_recebimento</code>)</h3>
<p>Valida a completude dos dados do beneficiário com base no tipo escolhido na coluna <code>tipo_recebimento</code>:</p>
<ul>
  <li>🪙 <strong>Se <code>WALLET_CRYPTO</code>:</strong> Exige endereço da carteira e rede blockchain.</li>
  <li>🇧🇷 <strong>Se <code>CHAVE_PIX</code>:</strong> Exige a chave Pix preenchida.</li>
  <li>🏦 <strong>Se <code>CONTA_BANCARIA</code>:</strong> Exige o preenchimento de todo o formulário SWIFT/Local (documento de identificação, titular, código do banco, agência, conta e tipo de conta).</li>
</ul>
