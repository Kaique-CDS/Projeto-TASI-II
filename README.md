# API de Gerenciamento de Pedidos

Projeto prático da disciplina de **Tópicos Avançados em Sistemas de Informação II**.
O objetivo é criar uma API REST em Node.js com Express para controlar os pedidos de uma loja, separando o código em camadas e aplicando validações e testes automatizados.

## Como instalar e rodar

1. Clone o repositório e instale as dependências:
```bash
npm install
```

2. Para iniciar o servidor:
```bash
npm start
```
A API vai rodar em `http://localhost:3000`.

3. Para rodar os testes automatizados (Jest):
```bash
npm test
```

## Arquitetura e organização do projeto

O projeto foi dividido no padrão de camadas pedido na atividade (`routes`, `controllers` e `services`), utilizando armazenamento em memória (array):

- `src/routes/pedidosRoutes.js`: define os endpoints da API e chama o controller.
- `src/controllers/pedidosController.js`: recebe a requisição HTTP (`req`, `res`), pega os parâmetros/body e devolve o status code correto. Se der erro, passa para o `next(err)`.
- `src/services/pedidosService.js`: concentra toda a regra de negócio, as validações dos campos, o controle das transições de status e a lista de pedidos em memória.
- `src/app.js` e `src/server.js`: o `app.js` configura o Express, as rotas e o middleware de erro, ficando separado do `server.js` (que só dá o `app.listen`) para facilitar os testes com o Supertest sem precisar subir a porta 3000 toda vez.
- `tests/pedidos.test.js`: testes automatizados das rotas e regras usando Jest e Supertest.
- `evidencias/`: pasta com os prints dos testes feitos no Postman.

## Regras de validação e valores aceitos

Para cadastrar um pedido (`POST /pedidos`), todos os campos abaixo são obrigatórios:

- **cliente**: texto não vazio (o sistema usa `.trim()` para não aceitar apenas espaços em branco).
- **valor**: número maior que zero.
- **tipo**: aceita `"entrega"` ou `"retirada"`.
- **prioridade**: aceita `"baixa"`, `"normal"`, `"alta"` ou `"urgente"`.

Quando o pedido é criado, ele recebe um `id` numérico sequencial e o `status` inicial é sempre `"recebido"`.

### Regras de transição de status

Os status possíveis são: `recebido`, `em_preparacao`, `enviado`, `entregue` e `cancelado`.
Para evitar mudanças sem sentido (como cancelar um pedido que já foi entregue ou entregar sem preparar), a atualização (`PATCH /pedidos/:id/status`) segue estas regras:

- De `recebido` -> pode ir para `em_preparacao` ou `cancelado`.
- De `em_preparacao` -> pode ir para `enviado`, `entregue` (caso seja retirada no balcão) ou `cancelado`.
- De `enviado` -> pode ir para `entregue` ou `cancelado`.
- De `entregue` ou `cancelado` -> não pode mais mudar de status (estados finais).

Se tentar fazer uma transição fora dessa ordem ou passar um status que não existe, a API retorna **HTTP 400** explicando o motivo no JSON.

## Rotas da API e Exemplos

### 1. Criar pedido
`POST /pedidos`

**Body (JSON):**
```json
{
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal"
}
```

**Resposta (201 Created):**
```json
{
  "id": 1,
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal",
  "status": "recebido"
}
```

### 2. Listar todos os pedidos
`GET /pedidos`

**Resposta (200 OK):**
```json
[
  {
    "id": 1,
    "cliente": "Carlos Silva",
    "tipo": "entrega",
    "valor": 350,
    "prioridade": "normal",
    "status": "recebido"
  }
]
```

### 3. Buscar pedido por ID
`GET /pedidos/:id`

- Se encontrar o ID: retorna **200 OK** com o objeto do pedido.
- Se não existir: retorna **404 Not Found** com `{"erro": "Pedido nao encontrado."}`.

### 4. Atualizar status do pedido
`PATCH /pedidos/:id/status`

**Body (JSON):**
```json
{
  "status": "cancelado"
}
```

- Se a transição for válida: retorna **200 OK** com o pedido atualizado.
- Se o status ou a transição forem inválidos: retorna **400 Bad Request**.
- Se o ID não existir: retorna **404 Not Found**.

## Uso de IA

- **Utilizou IA?** Sim, utilizei o ChatGPT/assistente de IA como apoio durante o desenvolvimento.
- **Para quais tarefas?** Usei para tirar dúvidas sobre como configurar o Jest junto com o Supertest no Express e para revisar a lógica de validação de transição de status no `pedidosService.js`.
- **Sugestão aceita, modificada ou rejeitada:**
  - **Sugestão modificada/rejeitada:** A IA inicialmente sugeriu criar pastas extras (`errors/` com uma classe `AppError` usando herança de `Error` e outra pasta `middlewares/`) além de vários arquivos separados. Eu rejeitei essa estrutura mais complexa e simplifiquei para uma função auxiliar `criarErro(mensagem, status)` dentro do próprio `pedidosService.js` e um middleware de erro direto no `app.js`. **Justificativa:** Como o projeto é pequeno e o enunciado pedia foco em `routes`, `controllers` e `services`, manter o tratamento de erro direto no `app.js` deixou o código bem mais fácil de ler, entender e explicar na avaliação, continuando a separar erros de validação (400/404) de erros inesperados do servidor (500).
