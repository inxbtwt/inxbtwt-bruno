# Bruno — inxbtwt-api

Abra esta pasta no Bruno e selecione **Local HTTPS**. Mantenha o Cookie Jar habilitado: os JWTs são enviados e renovados por cookies HttpOnly, Secure e SameSite=Strict.

## Endereço da API

- **Local HTTPS**: `https://localhost:8081`, destinado ao fluxo de autenticação por cookies. Configure HTTPS na API ou em um proxy e ajuste `baseUrl` se necessário. O backend ainda não possui uma configuração permanente de TLS. Use um certificado confiável para o seu computador.
- **Local**: `http://localhost:8081`, para diagnóstico dos endpoints HTTP. Cookies Secure não são enviados normalmente por clientes HTTP; este ambiente não é o fluxo recomendado de autenticação.

Inicie PostgreSQL e SeaweedFS para o fluxo completo de usuários e fotos. Não desabilite a validação de certificados globalmente.

## Ordem de execução

1. **Preparação > Obter CSRF**: guarda `csrfToken` e `csrfHeaderName`; o Cookie Jar armazena o cookie correspondente.
2. **Usuários > Cadastro > Criar usuário**: cria um usuário com endereço e guarda `loginEmail`, `userId` e `lastUser`.
3. **Usuários > Autenticação > Login**: envia e-mail e senha. A resposta usa ApiResponse, com `data.expiresIn`; os JWTs chegam exclusivamente em Set-Cookie.
4. **Usuário autenticado**: consulta `GET /me` e atualiza os dados usados nas demais requisições.
5. **Renovar tokens**: envia o cookie de refresh e o header CSRF, sem body. O servidor invalida o refresh anterior e substitui ambos os cookies.
6. **Confirmar sessão renovada**: consulta `/me` com o novo cookie de acesso.
7. Execute consultas, atualizações e fotos conforme as permissões configuradas no backend.

Para uma conta existente, execute Obter CSRF, defina `loginEmail` e `loginPassword` no ambiente e execute Login. A variável de runtime `loginEmail` criada pelo cadastro tem precedência durante a sessão; limpe-a para trocar de conta.

O script da collection injeta o header CSRF automaticamente em POST, PUT, PATCH e DELETE. Se apagar os cookies ou reiniciar a sessão, execute Obter CSRF novamente. Os JWTs não são copiados para variáveis nem enviados em Authorization. O Bruno gerencia os cookies, seus caminhos e expirações; o refresh pode ser executado mesmo com o cookie de acesso vencido.

## Permissões atuais

O YAML do backend libera cadastro, login, CSRF e refresh; `/me` exige autenticação. As consultas de usuários aceitam USER ou ADMIN e PUT exige autenticação. PATCH e upload de fotos ainda não estão liberados no YAML e retornarão 403 até que suas regras sejam adicionadas. Essas requisições foram mantidas para uso após a configuração das permissões.

## Usuários e arquivos

O cadastro aceita `addresses` com street, city, state, zipCode, country, referenceName, number e isDefault. As respostas incluem UUID e datas dos endereços, sem ID interno ou referência circular ao usuário. PUT e PATCH preservam os endereços existentes.

`fixtures/avatar.png` é usado nos uploads separados de perfil e capa. O cadastro gera um e-mail único e usa a senha fictícia `loginPassword`. O fluxo grava dados reais no banco local. As variáveis de runtime não persistem após reiniciar o Bruno.

As verificações existentes nas requisições de usuário foram preservadas; não foram adicionados novos blocos de testes automatizados.
