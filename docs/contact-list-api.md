# Contact List API — Documentação e plano de testes

API REST pública feita para estudo de testes, com usuários, autenticação por token (JWT) e CRUD de contatos. Todas as requisições e respostas usam JSON.

- **Base URL:** `https://thinking-tester-contact-list.herokuapp.com`
- **Documentação oficial:** https://documenter.getpostman.com/view/4012288/TzK2bEa8
- **App web:** https://thinking-tester-contact-list.herokuapp.com
- **Testes:** [`test/contact_list.spec.ts`](../test/contact_list.spec.ts)

## Como rodar

```bash
npm install
npm run test:contact
```

O relatório HTML é gerado em `output/report.html`.

Os testes **não usam uma conta fixa**. A cada execução, criam um usuário com dados aleatórios (faker) e o excluem no final. Nenhuma credencial fica no repositório.

## Autenticação

O cadastro (`POST /users`) e o login (`POST /users/login`) devolvem um `token`. As rotas protegidas exigem esse token no header:

```
Authorization: Bearer <token>
```

Sem token, com token inválido ou com token de uma sessão encerrada por logout, a resposta é:

```json
// 401 Unauthorized
{ "error": "Please authenticate." }
```

## Endpoints

### Usuários

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| POST | `/users` | não | Cadastra usuário e retorna token |
| POST | `/users/login` | não | Faz login e retorna token |
| GET | `/users/me` | sim | Retorna o perfil do usuário logado |
| PATCH | `/users/me` | sim | Atualiza campos do usuário |
| POST | `/users/logout` | sim | Invalida o token atual |
| DELETE | `/users/me` | sim | Exclui a conta |

**POST /users** (a mesma resposta vale para o login)

```json
// Request
{
  "firstName": "Ana",
  "lastName": "Silva",
  "email": "ana@teste.com",
  "password": "senha1234"
}

// 201 Created
{
  "user": {
    "_id": "6abb19c7d80def00157da8f3",
    "firstName": "Ana",
    "lastName": "Silva",
    "email": "ana@teste.com",
    "__v": 1
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Erro de validação** (padrão Mongoose, com um item em `errors` por campo inválido):

```json
// 400 Bad Request
{
  "errors": {
    "password": {
      "name": "ValidatorError",
      "message": "Path `password` is required.",
      "kind": "required",
      "path": "password"
    }
  },
  "_message": "User validation failed",
  "message": "User validation failed: password: Path `password` is required."
}
```

### Contatos

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| POST | `/contacts` | sim | Cadastra contato |
| GET | `/contacts` | sim | Lista os contatos do usuário logado |
| GET | `/contacts/:id` | sim | Busca um contato |
| PUT | `/contacts/:id` | sim | Substitui o contato inteiro |
| PATCH | `/contacts/:id` | sim | Atualiza só os campos enviados |
| DELETE | `/contacts/:id` | sim | Exclui o contato |

**POST /contacts**

```json
// Request
{
  "firstName": "João",
  "lastName": "Souza",
  "birthdate": "1990-05-10",
  "email": "joao@teste.com",
  "phone": "8005551234",
  "street1": "Rua A",
  "street2": "Apto 1",
  "city": "Florianópolis",
  "stateProvince": "SC",
  "postalCode": "88000",
  "country": "Brasil"
}

// 201 Created: mesmo objeto, com "_id" e "owner" (id do usuário dono)
```

## Regras de negócio

| # | Regra | Resposta |
|---|---|---|
| RN01 | `firstName`, `lastName` e `password` são obrigatórios no cadastro de usuário | 400, `kind: required` |
| RN02 | O e-mail do usuário é único | 400, `Email address is already in use` |
| RN03 | O e-mail precisa ter formato válido (usuário e contato) | 400, `Email is invalid` |
| RN04 | A senha tem entre 7 e 100 caracteres | 400, `minlength` / `maxlength` |
| RN05 | Nome e sobrenome têm no máximo 20 caracteres | 400, `maxlength` |
| RN06 | Credenciais inválidas ou incompletas no login | 401 |
| RN07 | Rotas de perfil e de contatos exigem token válido | 401, `Please authenticate.` |
| RN08 | O perfil nunca expõe a senha | campo `password` ausente |
| RN09 | `firstName` e `lastName` são obrigatórios no contato | 400, `kind: required` |
| RN10 | Telefone, data de nascimento (`AAAA-MM-DD`) e CEP são validados | 400, `... is invalid` |
| RN11 | Id de contato em formato inválido | 400, `Invalid Contact ID` |
| RN12 | Contato inexistente | 404 |
| RN13 | `PUT` exige o objeto completo; `PATCH` aceita atualização parcial | 400 / 200 |
| RN14 | Um usuário não vê nem acessa contatos de outro | 404 e lista vazia |
| RN15 | Depois do logout, o token deixa de valer | 401 |
| RN16 | Depois de excluir a conta, o login falha | 401 |

## Casos de teste

| Grupo | Caso | Esperado | Regra |
|---|---|---|---|
| Cadastro | Corpo vazio | 400 com erros de `firstName`, `lastName` e `password` | RN01 |
| Cadastro | E-mail já cadastrado | 400 | RN02 |
| Cadastro | E-mail inválido | 400 | RN03 |
| Cadastro | Senha com 3 caracteres | 400 | RN04 |
| Cadastro | Senha com 101 caracteres | 400 | RN04 |
| Cadastro | Nome com 21 caracteres | 400 | RN05 |
| Login | Credenciais válidas | 200 com token (JSON Schema validado) | — |
| Login | Senha errada | 401 | RN06 |
| Login | Sem senha | 401 | RN06 |
| Login | E-mail não cadastrado | 401 | RN06 |
| Perfil | `GET /users/me` | 200, sem campo `password` | RN08 |
| Perfil | Sem token | 401 | RN07 |
| Perfil | Token inválido | 401 | RN07 |
| Perfil | `PATCH` do nome | 200 com o nome novo | — |
| Perfil | `PATCH` com e-mail inválido | 400 | RN03 |
| Contatos | Cadastro completo | 201, `owner` igual ao id do usuário | — |
| Contatos | Sem nome e sobrenome | 400 | RN09 |
| Contatos | E-mail inválido | 400 | RN03 |
| Contatos | Telefone inválido | 400 | RN10 |
| Contatos | Data `10/05/1990` | 400 | RN10 |
| Contatos | CEP inválido | 400 | RN10 |
| Contatos | Sem token | 401 | RN07 |
| Contatos | Listagem | 200, array com o contato criado | — |
| Contatos | Busca por id | 200 | — |
| Contatos | Id inválido | 400 | RN11 |
| Contatos | Id inexistente | 404 | RN12 |
| Contatos | `PUT` parcial | 400 | RN13 |
| Contatos | `PUT` completo | 200 com todos os dados novos | RN13 |
| Contatos | `PATCH` de um campo | 200, só esse campo muda | RN13 |
| Contatos | Acesso por outro usuário | 404 e lista vazia | RN14 |
| Contatos | Exclusão | 200, `Contact deleted` | — |
| Contatos | Busca após exclusão | 404 | RN12 |
| Sessão | Token após logout | 401 | RN15 |
| Sessão | Login após excluir a conta | 401 | RN16 |

**Total: 34 testes.** Os `401` de login e o `404` de contato inexistente voltam com o corpo vazio, então nesses casos os testes validam só o status.

## Suíte da conta do dashboard

Arquivo separado: [`test/contact_list_dashboard.spec.ts`](../test/contact_list_dashboard.spec.ts). Ele usa uma **conta real**, então os contatos criados aparecem em https://thinking-tester-contact-list.herokuapp.com/contactList.

- **Só cria e consulta.** Não exclui contato nem conta, não edita dados e não faz logout.
- **Tem config e relatório próprios:** [`jest.dashboard.config.js`](../jest.dashboard.config.js) gera `output/contact-list-dashboard.html`.
- **Fica fora do `npm test`,** porque o `jest.config.js` ignora esse arquivo. A suíte principal não é afetada.
- **Credenciais:** `CONTACT_EMAIL` e `CONTACT_PASSWORD`, no `.env` (local) ou nos secrets do repositório (CI). Sem elas, os testes são pulados.

```bash
npm run test:dashboard
```

| Grupo | Caso | Esperado |
|---|---|---|
| POST | Contato com todos os campos | 201; fica salvo na conta |
| POST | Contato só com nome e sobrenome | 201; fica salvo na conta |
| POST | Sem campos obrigatórios | 400 (nada é criado) |
| POST | E-mail, telefone, data ou CEP inválido (4 casos) | 400 (nada é criado) |
| POST | Sem token | 401 |
| GET | Contato completo por id | 200 com os mesmos dados enviados |
| GET | Contato mínimo por id | 200, sem campos opcionais |
| GET | Lista de contatos | 200, inclui os dois criados |
| GET | Id inválido | 400 |
| GET | Id inexistente | 404 |
| GET | Sem token | 401 |
| GET | Perfil da conta | 200, sem campo `password` |

Cada execução adiciona **2 contatos** à conta: um completo (`street2` = "Teste automatizado") e um mínimo (sobrenome "Teste automatizado").
