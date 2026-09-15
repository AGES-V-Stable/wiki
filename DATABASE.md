<h1>🗄️ Documentação do Banco de Dados: V-Stable</h1>

<h2>🏗️ Visão Geral da Arquitetura</h2>
<p>O banco de dados da V-Stable foi modelado em <strong>PostgreSQL</strong>.</p>

<div align="center">
  <img width="100%" alt="modelagem_logica_vstable" src="https://github.com/user-attachments/assets/1e474744-2310-4457-af00-7dd8ff5db672" />
  <br/>
  <em>Modelo Lógico de Dados</em>
</div>

<div align="center">
  <img width="1400" style="max-width: none; width: 1400px; height: auto;" alt="ER Diagram0" src="https://github.com/user-attachments/assets/2ea951f3-0362-4cc1-8d0a-fe2e2d40e271" />
  <br/>
  <em>Modelo Conceitual</em>
</div>

<h3>Padrões Adotados</h3>
<ul>
  <li><strong>Identificadores:</strong> Utilização de <code>UUID v4</code> nativo para todas as chaves primárias (PK), garantindo segurança e impossibilitando a enumeração de registros em rotas [...]
  <li><strong>Padronização via ENUMs:</strong> Controle rígido de estados (Status de Transação, Compliance, Etapas de Onboarding, Métodos de Transferência e Redes Blockchain) diretamente no mot[...]
  <li><strong>Herança 1-para-1 (Table-per-Type):</strong> Separação elegante do fluxo de transações. Os dados financeiros comuns ficam na tabela <code>transacoes_base</code>, enquanto campos espe[...]
</ul>

<hr>

<h2>📖 Dicionário de Tabelas</h2>

<h3>1. Núcleo e Identidade (Core)</h3>
<p><em>*Nota Arquitetural: Para otimização do fluxo de cadastro (MVP), as entidades de Empresa e Usuário nascem simultaneamente na primeira etapa do processo.</em></p>
<ul>
  <li><strong><code>empresas</code>:</strong> Tabela central do sistema. Armazena os dados cadastrais da PME, endereço, informações de compliance (propósito, receita) e o saldo disponível na plat[...]
    <blockquote><strong>💡 Destaque:</strong> Possui controle de aprovação de compliance granularizado por moeda (<code>kyc_usd_aprovado</code> e <code>kyc_cop_aprovado</code>). Campos necessário[...]
  </li>
  <li><strong><code>usuarios</code>:</strong> Representantes legais que operam o painel da PME.
    <blockquote><strong>💡 Destaque:</strong> Centraliza a segurança da conta (credenciais, status de atividade e <code>segredo_2fa</code> para validação TOTP no backend). Exige obrigatoriamente [...]
  </li>
  <li><strong><code>administradores</code>:</strong> Usuários internos da V-Stable (staff). Tabela isolada das PMEs, com níveis de acesso definidos (<code>SUPER_ADMIN</code>, <code>ANALISTA_COMPLIAN[...]
</ul>

<h3>2. Onboarding e Compliance</h3>
<ul>
  <li><strong><code>progresso_cadastros</code>:</strong> Tracker de navegação e pendências (Checklist). Como a conta oficial já existe desde o primeiro passo, esta tabela rastreia as etapas subseq[...]
    <blockquote><strong>💡 Destaque:</strong> A arquitetura evoluiu e não utiliza mais JSONB temporário. Agora é vinculada via <code>usuario_id</code> e utiliza o campo <code>etapa_pendente</code[...]
  </li>
  <li><strong><code>documentos_compliance</code>:</strong> Registro dos arquivos (Contrato Social, Comprovantes de Endereço, CNH) enviados para auditoria. Armazena a URL do bucket S3 e o status de ap[...]
</ul>

<h3>3. Diretório Financeiro</h3>
<ul>
  <li><strong><code>beneficiarios</code>:</strong> Agenda de contatos unificada por PME. Suporta múltiplos destinos financeiros em um único registro.
    <blockquote><strong>💡 Destaque:</strong> Registra identificadores da API parceira (<code>avenia_id</code> para contas fiduciárias e <code>avenia_wallet_id</code> para carteiras). Suporta conta[...]
  </li>
</ul>

<h3>4. Motor de Transações (Herança Table-per-Type)</h3>
<ul>
  <li><strong><code>transacoes_base</code>:</strong> Concentra o núcleo monetário de qualquer movimentação (seja entrada ou saída).
    <blockquote><strong>💡 Destaque:</strong> Armazena a conversão exata da operação (<code>moeda_estrangeira</code>, <code>valor_estrangeiro</code>, <code>valor_liquidacao_brl</code>, <code>taxa[...]
  </li>
  <li><strong><code>transacoes_importacao</code>:</strong> Extensão da transação para fluxos de Saída (Pagamento de fornecedores).
    <blockquote><strong>💡 Destaque:</strong> A chave primária é a própria FK (<code>transacao_id</code>). Exige o vínculo obrigatório e direto com a tabela de <code>beneficiarios</code> e a de[...]
  </li>
  <li><strong><code>transacoes_exportacao</code>:</strong> Extensão da transação para fluxos de Entrada (Geração de Invoices/Cobranças Internacionais).
    <blockquote><strong>💡 Destaque:</strong> Totalmente desacoplada da tabela de beneficiários (já que o dinheiro entra). Armazena o link/código de pagamento único (<code>codigo_cobranca_extern[...]
  </li>
</ul>

<hr>

<h2>🛡️ Regras de Negócio e Integridade (<code>CHECK Constraints</code>)</h2>
<p>Para blindar o banco de dados contra inconsistências lógicas vindas do backend, foram implementadas restrições nativas no PostgreSQL:</p>

<h3>Isolamento Estrutural de Transações</h3>
<p>Em vez de depender de validações frágeis em uma única tabela gigantesca, a própria divisão física em <code>transacoes_importacao</code> (exigindo Beneficiário) e <code>transacoes_exportacao[...]

<h3>Consistência de Destinos Financeiros (<code>chk_dados_recebimento</code>)</h3>
<p>Valida a completude dos dados na tabela <code>beneficiarios</code> com base no tipo escolhido na coluna <code>tipo_recebimento</code>:</p>
<ul>
  <li>🪙 <strong>Se <code>WALLET_CRYPTO</code>:</strong> Exige obrigatoriamente o endereço da carteira e a rede blockchain.</li>
  <li>🇧🇷 <strong>Se <code>CHAVE_PIX</code>:</strong> Exige a chave Pix preenchida.</li>
  <li>🏦 <strong>Se <code>CONTA_BANCARIA</code>:</strong> Exige o preenchimento de todo o formulário SWIFT/Local (documento de identificação, titular, código do banco, agência, conta e tipo de [...]
</ul>
