# Especificação - Document Management System

> Especificação do MVP para orientar o desenvolvimento guiado por especificação. Os requisitos abaixo descrevem o comportamento a implementar; não significam que as funcionalidades já existam no seed atual.

## 1. Objetivo

Permitir que um usuário envie, liste e baixe documentos armazenados no filesystem local da aplicação, mantendo seus metadados em memória.

## 2. Escopo

### Dentro do escopo

- Enviar um documento por requisição.
- Listar os documentos associados à identidade local configurada.
- Baixar um documento pelo identificador.
- Registrar metadados de cada documento em memória durante a execução do processo.
- Usar Multer com `diskStorage` para gravar os arquivos em armazenamento local.
- Exibir, no frontend React, a lista e as ações de envio e download, com estados de carregamento e erro.

### Fora do escopo

- Armazenamento externo ou em nuvem.
- Versionamento, edição, remoção ou compartilhamento de documentos.
- Banco de dados ou persistência durável dos metadados.
- Cadastro de usuários, login, autenticação e autorização multiusuário.
- Pré-visualização do conteúdo dos arquivos.
- Restrições por allowlist de tipos MIME ou extensões nesta versão.

### Limitações conhecidas do MVP

- A aplicação usa uma identidade local única, sem autenticação. O campo `owner` registra essa identidade, mas não representa uma conta autenticada.
- Os metadados existem somente em memória. Ao reiniciar o backend, a lista é perdida embora os arquivos continuem em `backend/storage`; esses arquivos não serão recuperáveis pela API após o reinício.

## 3. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O sistema deve aceitar um arquivo enviado como `multipart/form-data` no campo `file`. | Uma requisição válida cria um documento e retorna seus metadados com status `201`. |
| RF-02 | O sistema deve rejeitar requisição sem arquivo, arquivo vazio ou multipart inválido. | A API retorna `400` com erro JSON; nenhum metadado de documento é criado. |
| RF-03 | O sistema deve limitar o tamanho do arquivo conforme configuração. | Arquivo acima do limite retorna `413`; o padrão é 10 MiB quando a configuração não é informada. |
| RF-04 | O sistema deve atribuir um identificador único e um proprietário local ao documento. | O ID não é derivado do nome enviado; `owner` corresponde à identidade local configurada. |
| RF-05 | O sistema deve gravar o conteúdo no diretório local configurado, usando nome físico gerado pelo sistema. | O nome original não é usado como caminho no filesystem; colisões e traversal por nome de arquivo não são possíveis. |
| RF-06 | O usuário local deve poder listar os metadados dos documentos disponíveis na execução atual. | A resposta contém todos e somente os documentos pertencentes à identidade local. A ordem é da data de upload mais recente para a mais antiga. |
| RF-07 | O usuário local deve poder baixar um documento pelo ID. | Um ID existente pertencente à identidade local retorna o conteúdo como anexo com o nome original. |
| RF-08 | O sistema deve tratar IDs inválidos ou documentos indisponíveis. | ID malformado retorna `400`; documento inexistente ou não pertencente ao usuário local retorna `404`. |
| RF-09 | O frontend deve permitir selecionar e enviar um arquivo, consultar a lista e iniciar o download. | A interface apresenta estados de carregamento e mensagens de erro sem perder a lista já carregada. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos devem ser armazenados exclusivamente no filesystem local, em `backend/storage` por padrão, utilizando Multer com `diskStorage`. |
| RNF-02 | Os metadados devem ser mantidos em memória nesta fase; não adicionar banco de dados ou armazenamento externo. A perda dos metadados no reinício é comportamento esperado do MVP. |
| RNF-03 | Porta, diretório de armazenamento, limite de upload e identidade local devem ser configuráveis por variáveis de ambiente, com valores padrão documentados. |
| RNF-04 | O backend deve manter a separação `routes -> controllers -> services -> repositories`; cada camada depende apenas da camada imediatamente interna. |
| RNF-05 | O backend deve validar entrada HTTP e tratar falhas de leitura/escrita, retornando respostas HTTP consistentes sem expor stack traces ou caminhos internos. |
| RNF-06 | Nomes físicos devem ser gerados pelo servidor e IDs validados antes de compor caminhos de arquivo. Nomes originais devem ser tratados como dados não confiáveis ao compor cabeçalhos de download. |
| RNF-07 | O frontend deve chamar a API com o prefixo `/api` configurado no proxy do Vite; o proxy encaminha ao backend sem esse prefixo. |
| RNF-08 | Usar JavaScript e as dependências já presentes no projeto sempre que possível; manter a solução simples e compatível com SOLID, DRY, KISS, YAGNI e 12-Factor. |

### Configuração proposta

| Variável | Padrão | Uso |
| --- | --- | --- |
| `PORT` | `3000` | Porta HTTP do backend, respeitando o valor já adotado pelo ambiente de desenvolvimento. |
| `STORAGE_DIR` | `backend/storage` | Diretório local de gravação dos arquivos. |
| `MAX_UPLOAD_SIZE_BYTES` | `10485760` | Limite de upload em bytes (10 MiB). |
| `DMS_DEFAULT_OWNER` | `local-user` | Identidade local atribuída aos documentos e usada na listagem/download. |

## 5. Modelo de dados (metadados do documento)

### Documento

| Campo | Tipo JSON | Descrição e regra |
| --- | --- | --- |
| `id` | string | UUID gerado pelo backend. Também identifica o nome físico do arquivo no diretório de armazenamento; não contém caminho fornecido pelo cliente. |
| `originalName` | string | Nome original fornecido no upload, preservado para exibição e download; nunca usado diretamente como caminho físico. |
| `size` | number | Tamanho do arquivo em bytes, inteiro não negativo. Arquivos vazios são rejeitados. |
| `uploadedAt` | string | Data/hora de recebimento em ISO 8601 UTC. |
| `owner` | string | Identidade local obtida da configuração do servidor, não do corpo da requisição. |

O objeto público de documento contém somente os cinco campos acima. O repositório mantém a relação entre `id`, arquivo físico e metadados. O arquivo não é serializado dentro do objeto JSON.

### Persistência

- O conteúdo binário é gravado em `STORAGE_DIR`, com nome físico derivado do UUID do documento.
- Os metadados ficam em uma coleção em memória pertencente ao repositório durante a vida do processo.
- Um erro ao gravar o arquivo não pode deixar metadados publicados. Se uma falha ocorrer após a gravação mas antes do registro, o arquivo parcial ou órfão deve ser removido quando possível.
- Não há recuperação ou reconstrução de metadados a partir dos arquivos depois de reiniciar o processo.

## 6. Contratos de API

Os caminhos nesta seção são os caminhos do backend. No frontend, as chamadas usam `/api` como prefixo do proxy do Vite: por exemplo, `/api/upload` é encaminhado para `/upload` no backend.

As respostas JSON de erro seguem o formato:

```json
{
  "error": {
    "code": "FILE_REQUIRED",
    "message": "Envie um arquivo no campo file."
  }
}
```

`code` é estável para uso pelo cliente; `message` é uma mensagem legível em português. Respostas de erro não incluem stack trace nem caminho local.

### POST /upload

- Entrada: `multipart/form-data`, com exatamente um arquivo no campo `file`.
- Limite: `MAX_UPLOAD_SIZE_BYTES`; arquivo vazio é inválido. Não há allowlist de MIME/extensão no MVP.
- Processamento: Multer `diskStorage` grava o arquivo com nome gerado pelo servidor. O backend cria ID, data e `owner`, registra os metadados e não confia em campos de proprietário enviados pelo cliente.
- Sucesso: `201 Created`, `Content-Type: application/json`, corpo é o objeto `Document`.
- Erros:
  - `400 Bad Request`, `FILE_REQUIRED`, `INVALID_MULTIPART` ou `EMPTY_FILE` para arquivo ausente, multipart inválido ou conteúdo vazio.
  - `413 Payload Too Large`, `FILE_TOO_LARGE` quando o arquivo ultrapassar o limite.
  - `500 Internal Server Error`, `STORAGE_ERROR` em falha de filesystem; detalhes internos são registrados somente no servidor.

### GET /documents

- Entrada: sem corpo.
- Sucesso: `200 OK`, `Content-Type: application/json`, corpo é um array de objetos `Document`, ordenado por `uploadedAt` decrescente. Se não houver documentos, retorna `[]`.
- A lista inclui somente documentos cujo `owner` corresponde à identidade local configurada.
- Erro: `500 Internal Server Error`, `DOCUMENTS_READ_ERROR`, se não for possível consultar o repositório em memória.

### GET /documents/:id/download

- Entrada: `id` UUID na rota; sem corpo.
- Sucesso: `200 OK`, conteúdo binário como anexo (`Content-Disposition: attachment`) com o nome original sanitizado no cabeçalho. Usar `application/octet-stream` como `Content-Type` para não depender de MIME informado pelo cliente.
- Erros:
  - `400 Bad Request`, `INVALID_DOCUMENT_ID`, quando o formato do ID for inválido.
  - `404 Not Found`, `DOCUMENT_NOT_FOUND`, se o ID não existir na memória atual ou não pertencer à identidade local.
  - `500 Internal Server Error`, `STORAGE_ERROR`, em falha ao ler o arquivo.

## 7. Decisões arquiteturais

- **Backend:** Node.js + Express em CommonJS. Manter as responsabilidades nas camadas existentes:
  - `routes/` declara caminhos e encaminha para controllers.
  - `controllers/` valida o formato básico da entrada HTTP e traduz resultados/erros para status e respostas.
  - `services/` aplica regras de negócio, validações e coordena as operações.
  - `repositories/` mantém metadados em memória e faz a leitura/gravação no filesystem local.
- **Upload:** usar Multer com `diskStorage`; destino e limite vêm da configuração. Não armazenar o binário em memória nem enviar arquivos a serviços externos.
- **Identificadores e caminhos:** gerar UUID no servidor; derivar o nome físico do ID. Validar o ID de download antes de acessar o filesystem.
- **Usuário:** sem autenticação no MVP. `DMS_DEFAULT_OWNER` é a identidade local única; não aceitar `owner` definido pelo cliente.
- **Frontend:** React com componentes funcionais e organização em `components/`, `pages/` e `services/`. A comunicação usa `fetch` pelo prefixo `/api` do proxy Vite.
- **Testes:** usar o runner nativo `node:test` no backend. Cobrir sucesso, validação, isolamento por owner, limite de tamanho, documento inexistente e falhas de armazenamento; validar também a integração da UI com a API.
- **Estado atual:** o seed contém somente o endpoint `/health` no backend, o frontend é placeholder e as rotas de documento ainda serão implementadas. Esta especificação não afirma que esses fluxos já estejam disponíveis.

## 8. Plano de execução

As etapas abaixo descrevem implementação futura do produto. Esta especificação não executa essas etapas nem altera arquivos de aplicação.

1. Preparar configuração e persistência local do backend: variáveis de ambiente, diretório de armazenamento, repositório de metadados em memória e convenções de erro.
2. Implementar o fluxo de upload: validação multipart e de tamanho, gravação Multer em disco, geração de metadados e limpeza em caso de falha; cobrir com testes unitários/HTTP.
3. Implementar listagem e download com filtro pela identidade local, validação de ID, cabeçalhos de anexo e tratamento de erros; cobrir os cenários de sucesso e falha.
4. Implementar no frontend a seleção/upload, listagem, download e estados de carregamento/erro, integrando via `/api`.
5. Executar testes backend, validar os fluxos ponta a ponta e revisar limites, configuração e documentação do MVP.
