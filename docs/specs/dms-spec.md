# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web simples para armazenar documentos no filesystem local, listar os documentos enviados e permitir seu download por identificador, usando metadados em memória e uma instalação single-user nesta fase.

## 2. Escopo

### Dentro do escopo

- Enviar um arquivo por requisição HTTP.
- Listar os metadados dos documentos recebidos durante a execução atual do backend.
- Baixar um documento pelo identificador gerado pela aplicação.
- Exibir no frontend os estados de envio, listagem vazia, sucesso e erro.
- Atribuir os documentos a um proprietário padrão, sem autenticação nesta fase.

### Fora do escopo

- Login, autenticação, autorização ou isolamento seguro entre usuários.
- Persistência dos metadados em banco de dados ou em outro processo.
- Armazenamento em nuvem, remoto ou por provedor de terceiros.
- Versionamento, exclusão, busca, filtros, ordenação escolhida pelo usuário ou paginação.
- Preview de arquivos, conversão, antivírus ou restrição por tipo de arquivo.
- Garantia de recuperação dos metadados após reiniciar o backend.

**Premissa de uso:** a primeira versão é single-user e destinada a um ambiente local/confiável. O campo `owner` é metadado descritivo, não um mecanismo de autenticação ou controle de acesso. Não se deve expor a API como serviço multiusuário ou publicamente acessível sem definir e implementar controles de segurança apropriados.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O sistema deve aceitar um arquivo no campo multipart `file` de `POST /upload`. Cada requisição aceita exatamente um arquivo. A ausência do campo deve resultar em `400`; arquivo acima do limite configurado deve resultar em `413`. Tipos de arquivo não são filtrados nesta fase. |
| RF-02 | Para cada upload aceito, o sistema deve gravar o conteúdo em `backend/storage` com um nome interno gerado pela aplicação, sem usar o nome original como caminho. Deve criar metadados com identificador único, nome original, tamanho em bytes, data/hora UTC em ISO 8601 e proprietário padrão `local-user`. |
| RF-03 | Em upload concluído, `POST /upload` deve responder `201 Created` com os metadados públicos do documento. Se a gravação ou o registro dos metadados falhar, deve remover qualquer arquivo parcial quando possível, não retornar sucesso e responder com erro `500`. |
| RF-04 | `GET /documents` deve responder `200 OK` com uma lista JSON de metadados dos documentos registrados na execução atual. A lista vazia deve ser `[]`. A ordem padrão é do mais recente para o mais antigo. |
| RF-05 | `GET /documents/:id/download` deve localizar o documento pelo identificador da aplicação e transmitir seu conteúdo como anexo. O nome original pode ser usado apenas no cabeçalho de download, nunca como caminho no filesystem. |
| RF-06 | Se o identificador não estiver registrado, ou se o arquivo correspondente não existir, o download deve responder `404 Not Found` com erro JSON. Caminhos internos e detalhes do filesystem não devem ser incluídos na resposta. |
| RF-07 | Os erros da API devem usar o formato JSON definido na seção 6, com código estável e mensagem em português. Erros inesperados devem ser tratados na borda HTTP e não devem expor stack traces ou caminhos locais. |
| RF-08 | O frontend deve permitir selecionar e enviar um arquivo, atualizar a listagem após sucesso e iniciar o download pelo identificador. Deve apresentar estados de carregamento, vazio, sucesso e erro sem bloquear o uso da tela. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os binários devem ser gravados exclusivamente no filesystem local, no diretório `backend/storage`, usando Multer com `diskStorage`. Não utilizar serviços ou provedores externos de armazenamento. |
| RNF-02 | Os metadados devem ser mantidos em memória nesta fase. Após reiniciar o backend, a lista e o vínculo entre metadados e arquivos deixam de estar disponíveis; os arquivos já gravados podem permanecer no diretório local sem metadados associados. |
| RNF-03 | O tamanho máximo padrão por arquivo deve ser 10 MiB (`10485760` bytes), configurável pela variável de ambiente `MAX_FILE_SIZE_BYTES`. O valor deve ser um inteiro positivo; configuração inválida deve ser reportada claramente durante a inicialização. |
| RNF-04 | O backend deve manter a configuração operacional em variáveis de ambiente, incluindo `PORT` (padrão atual `3000`) e `MAX_FILE_SIZE_BYTES`. A localização de armazenamento permanece fixa em `backend/storage` nesta fase. |
| RNF-05 | O nome original do arquivo é dado não confiável: não pode determinar o caminho de gravação e deve ser sanitizado/adequadamente codificado ao compor o cabeçalho `Content-Disposition`. O nome interno deve ser gerado pela aplicação e o acesso ao arquivo deve permanecer confinado ao diretório de armazenamento. |
| RNF-06 | Rotas e controllers devem tratar erros de entrada, Multer e filesystem com respostas HTTP consistentes. Respostas de erro não devem vazar detalhes internos. |
| RNF-07 | O backend deve seguir a separação `routes -> controllers -> services -> repositories`, com Node.js e Express em CommonJS e testes usando `node:test`. O frontend deve seguir React/Vite e comunicar-se com o backend por `fetch` usando o prefixo `/api` do proxy existente. |

## 5. Modelo de dados (metadados do documento)

### Representação pública

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único, gerado pela aplicação e usado nas rotas de listagem e download. |
| `originalName` | string | Nome original fornecido no upload, tratado como texto não confiável. |
| `size` | number | Tamanho do arquivo em bytes, inteiro não negativo. |
| `uploadedAt` | string | Data e hora do upload em ISO 8601, normalizada para UTC. |
| `owner` | string | Identificador descritivo do proprietário. Nesta versão tem o valor fixo `local-user`; não representa usuário autenticado. |

### Dados internos de persistência

O repositório mantém, além da representação pública, o nome interno gerado para o arquivo (por exemplo, `storedFilename`). Esse valor é necessário para localizar o binário em `backend/storage`, mas não deve ser retornado pela API. A associação dos metadados ao arquivo existe apenas em memória durante a execução do backend.

## 6. Contratos de API

Os caminhos abaixo são relativos ao backend. No frontend, as chamadas usam o prefixo `/api` (por exemplo, `/api/upload`), removido pelo proxy do Vite antes de encaminhar a requisição.

### Formato de erro

Todas as respostas de erro JSON devem seguir este formato:

```json
{
  "error": {
    "code": "FILE_REQUIRED",
    "message": "Envie um arquivo no campo 'file'."
  }
}
```

`code` é estável para uso pelo frontend; `message` é destinada à apresentação em português. Respostas de download bem-sucedidas são binárias, não JSON.

### `POST /upload`

- **Content-Type:** `multipart/form-data`
- **Campo obrigatório:** `file`, contendo um arquivo.
- **Limite:** `MAX_FILE_SIZE_BYTES`; padrão `10485760` bytes.
- **Tipos aceitos:** qualquer tipo nesta fase.
- **Sucesso:** `201 Created`, `Content-Type: application/json`.

Exemplo de resposta:

```json
{
  "id": "0f4e3b15-3f36-4c73-9b62-bcdfc5f8c39a",
  "originalName": "relatorio.pdf",
  "size": 48210,
  "uploadedAt": "2026-11-11T14:30:00.000Z",
  "owner": "local-user"
}
```

Erros previstos:

| Status | Código | Condição |
| --- | --- | --- |
| `400 Bad Request` | `FILE_REQUIRED` | Campo `file` ausente. |
| `400 Bad Request` | `INVALID_MULTIPART` | Requisição multipart inválida ou campos/arquivos inesperados. |
| `413 Payload Too Large` | `FILE_TOO_LARGE` | Arquivo ultrapassa o limite configurado. |
| `500 Internal Server Error` | `UPLOAD_FAILED` | Falha inesperada ao gravar o arquivo ou registrar os metadados. |

### `GET /documents`

- **Entrada:** sem parâmetros ou corpo.
- **Sucesso:** `200 OK`, `Content-Type: application/json`.
- **Resposta:** array de metadados públicos, ordenado por `uploadedAt` decrescente. Se não houver documentos registrados, retornar `[]`.
- **Erro:** `500 Internal Server Error` com código `LIST_FAILED` se ocorrer uma falha inesperada ao consultar o repositório.

Exemplo de resposta:

```json
[
  {
    "id": "0f4e3b15-3f36-4c73-9b62-bcdfc5f8c39a",
    "originalName": "relatorio.pdf",
    "size": 48210,
    "uploadedAt": "2026-11-11T14:30:00.000Z",
    "owner": "local-user"
  }
]
```

### `GET /documents/:id/download`

- **Entrada:** `id` gerado pela aplicação no segmento da rota.
- **Sucesso:** `200 OK`, conteúdo binário como anexo, `Content-Type: application/octet-stream` e `Content-Disposition: attachment` com o nome original devidamente sanitizado/codificado.
- **Erros previstos:**

| Status | Código | Condição |
| --- | --- | --- |
| `404 Not Found` | `DOCUMENT_NOT_FOUND` | ID desconhecido ou arquivo correspondente ausente do armazenamento local. |
| `500 Internal Server Error` | `DOWNLOAD_FAILED` | Falha inesperada ao ler ou transmitir o arquivo. |

O caminho interno nunca deve ser retornado ao cliente. A resposta de erro deve seguir o formato JSON comum quando ainda não tiver começado a transmissão do binário.

## 7. Decisões arquiteturais

- **Backend:** Node.js + Express, CommonJS, JavaScript. O fluxo de dependência é `routes -> controllers -> services -> repositories`.
- **Routes:** registram os endpoints e associam middlewares/controllers; não contêm regras de negócio.
- **Multer:** middleware de entrada do upload configurado com `diskStorage`, diretório local `backend/storage`, nome interno gerado e limite de tamanho. O nome original nunca define o destino de gravação.
- **Controllers:** fazem a tradução entre HTTP e aplicação, validam a forma básica da entrada, chamam os services e convertem resultados/erros para status e formatos HTTP.
- **Services:** aplicam as regras de upload, listagem e download sem depender de Express ou de objetos de request/response.
- **Repositories:** mantêm os metadados em memória e resolvem o vínculo com os arquivos locais. Arquivos e metadados não devem ser tratados como persistência durável nesta fase.
- **Frontend:** React + Vite, componentes funcionais e `fetch` em um serviço reutilizável. As chamadas usam `/api`; o proxy existente encaminha para o backend.
- **Configuração:** variáveis de ambiente para porta e limite de tamanho; sem serviço externo. O diretório local é `backend/storage`.
- **Testes:** runner nativo `node:test` para serviços e contratos HTTP, incluindo caminhos de sucesso e falhas previstas.

## 8. Plano de execução

As etapas abaixo descrevem trabalho futuro. Para esta entrega, somente este documento de especificação deve ser criado; não implementar nem alterar arquivos de execução do backend ou frontend.

1. Escrever testes backend com `node:test` para upload válido, arquivo ausente, limite excedido, listagem vazia/preenchida, download válido e ID/arquivo inexistente.
2. Implementar a configuração de armazenamento local com Multer `diskStorage`, limite de tamanho, nome interno gerado e tratamento de falha/limpeza de arquivos parciais.
3. Implementar o repositório em memória e os services de upload, listagem e download, mantendo os caminhos locais fora da representação pública.
4. Implementar controllers e rotas para os três contratos, mapear erros para os status e envelopes definidos e registrar as rotas no app Express.
5. Implementar no frontend o serviço `fetch`, o formulário de upload, a listagem e a ação de download, incluindo estados de carregamento, vazio e erro.
6. Executar os testes do backend e o build do frontend; validar manualmente o fluxo upload -> listagem -> download e os principais casos de erro, confirmando que os arquivos ficam somente em `backend/storage`.
