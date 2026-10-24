# Bruno — inxbtwt-api

Abra esta pasta no Bruno e selecione **Local HTTPS**. Mantenha o Cookie Jar habilitado: os JWTs são enviados e renovados por cookies HttpOnly, Secure e SameSite=Strict.

## Endereço da API

- **Local HTTPS**: `https://localhost:8081`, destinado ao fluxo de autenticação por cookies. Configure HTTPS na API ou em um proxy e ajuste `baseUrl` se necessário. O backend ainda não possui uma configuração permanente de TLS. Use um certificado confiável para o seu computador.
- **Local**: `http://localhost:8081`, para diagnóstico dos endpoints HTTP. Cookies Secure não são enviados normalmente por clientes HTTP; este ambiente não é o fluxo recomendado de autenticação.

Inicie PostgreSQL e SeaweedFS para o fluxo completo de usuários e fotos. Não desabilite a validação de certificados globalmente.

## Ordem de execução

1. **Preparação > Obter CSRF**: guarda `csrfToken` e `csrfHeaderName`; o Cookie Jar armazena o cookie correspondente.
2. **Usuários > Cadastro > Criar usuário**: cria um usuário com username, e-mail e telefone únicos e endereço; guarda `loginEmail`, `userId` e `lastUser`.
3. **Usuários > Autenticação > Login**: envia e-mail e senha. A resposta usa ApiResponse, com `data.expiresIn`; os JWTs chegam exclusivamente em Set-Cookie.
4. **Usuário autenticado**: consulta `GET /me` e atualiza os dados usados nas demais requisições.
5. **Renovar tokens**: envia o cookie de refresh e o header CSRF, sem body. O servidor invalida o refresh anterior e substitui ambos os cookies.
6. **Confirmar sessão renovada**: consulta `/me` com o novo cookie de acesso.
7. Execute consultas, atualizações e fotos. O upload de capa usa `POST /users/{userId}/photo?type=COVER` com o arquivo no campo multipart `photo`.
8. **Usuários > Perfil > Atualizar bio do perfil**: salva a bio em `PATCH /profiles/me`, preservando a capa. A primeira gravação cria o perfil.
9. **Consultar perfil autenticado**: confirma bio e `coverUrl`. Para baixar a capa, use **Fotos > Consultar URLs das fotos** e **Buscar foto da capa**.
10. **Logout**: encerra a sessão e limpa as variáveis do usuário na collection.

Para uma conta existente, execute Obter CSRF, defina `loginEmail` e `loginPassword` no ambiente e execute Login. A variável de runtime `loginEmail` criada pelo cadastro tem precedência durante a sessão; limpe-a para trocar de conta.

O script da collection injeta o header CSRF automaticamente em POST, PUT, PATCH e DELETE. Se apagar os cookies ou reiniciar a sessão, execute Obter CSRF novamente. Os JWTs não são copiados para variáveis nem enviados em Authorization. O Bruno gerencia os cookies, seus caminhos e expirações; o refresh pode ser executado mesmo com o cookie de acesso vencido.

## Permissões atuais

O YAML libera cadastro, login, logout, CSRF, refresh e `GET /users/availability`. `/me`, PUT/PATCH de usuário, upload de fotos e GET/PATCH de `/profiles/me` exigem autenticação. As consultas de usuários e fotos aceitam USER ou ADMIN.

## Usuários e arquivos

O cadastro exige `username`, além de nome, e-mail, telefone e senha. Username e e-mail são únicos sem distinção de maiúsculas; telefone é único pela sequência de dígitos, ignorando a formatação. O cadastro aceita `addresses` com street, city, state, zipCode, country, referenceName, number e isDefault. As respostas incluem UUID e datas dos endereços, sem ID interno ou referência circular ao usuário. PUT e PATCH preservam os endereços existentes.

`fixtures/avatar.png` é usado nos uploads separados de perfil e capa. A capa enviada é persistida em `profile_entity.cover_url`, e a foto de perfil permanece no usuário. O cadastro gera username, e-mail e telefone únicos e usa a senha fictícia `loginPassword`. O fluxo grava dados reais no banco local. As variáveis de runtime não persistem após reiniciar o Bruno.

## Disponibilidade

As três requisições da pasta **Disponibilidade** usam `/users/availability?field=...&value=...`, com `field` igual a `username`, `email` ou `phone`. Configure `availabilityUsername`, `availabilityEmail` e `availabilityPhone` no ambiente para conferir valores livres ou já utilizados. A resposta mantém `data.available`.

As novas requisições são destinadas à execução e conferência manual, sem novos blocos de testes automatizados. As verificações antigas de cadastro e PUT foram removidas porque dependiam de dados fixos incompatíveis com os campos únicos.
