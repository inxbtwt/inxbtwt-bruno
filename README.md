# Collection Bruno — inxbtwt-api

No Bruno, selecione **Open Collection** e abra esta pasta (`Documentos/bruno/inxbtwt-api`).

O endereço padrão é `http://localhost:8080`, definido em `collection.bru`. O ambiente **Local** também está disponível; nenhuma credencial externa é necessária.

Com a API e o PostgreSQL iniciados, execute a pasta **users** pelo Collection Runner. As requisições estão ordenadas:

1. Criar usuário: gera um e-mail único e salva `userId` e os dados retornados em variáveis de runtime.
2. Consultar usuário: reutiliza o UUID e verifica os dados criados.
3. Listar usuários: verifica a paginação.
4. Atualizar usuário (PUT): envia todos os campos obrigatórios.
5. Atualizar usuário parcialmente (PATCH): altera o nome e verifica que os demais campos foram preservados.
6. Confirmar alterações persistidas: consulta novamente e compara os dados.

Também é possível enviar uma requisição por vez; execute **Criar usuário** primeiro para preencher as variáveis. Uma nova execução da criação gera outro e-mail, evitando conflitos. As variáveis de runtime não são persistidas entre reinicializações do Bruno.

Para executar pelo terminal nesta pasta:

```powershell
npx --yes @usebruno/cli@latest run --env Local --sandbox safe
```

O fluxo cria um usuário real no banco local e atualiza apenas esse usuário. O registro permanece no banco ao terminar, pois a API não tem endpoint DELETE.

- Sucesso: HTTP `201`, com `ApiResponse<UserResponse>`.
- Dados inválidos: HTTP `400`, com mensagem em `ApiResponse`.
- E-mail duplicado: HTTP `409`; o fluxo automatizado gera e-mails únicos para evitar esse caso.

A senha é fictícia para desenvolvimento. A URL da foto é um exemplo e não implica que o arquivo exista no storage.
