# Bruno — inxbtwt-api

Abra esta pasta no Bruno e selecione **Local HTTPS**. Mantenha o Cookie Jar habilitado: os JWTs são enviados e renovados por cookies HttpOnly, Secure e SameSite=Strict.

## Endereço da API

- **Local HTTPS**: `https://localhost:8081`, destinado ao fluxo de autenticação por cookies. Configure HTTPS na API ou em um proxy e ajuste `baseUrl` se necessário. O backend ainda não possui uma configuração permanente de TLS. Use um certificado confiável para o seu computador.
- **Local**: `http://localhost:8081`, para diagnóstico dos endpoints HTTP. Cookies Secure não são enviados normalmente por clientes HTTP; este ambiente não é o fluxo recomendado de autenticação.

Inicie PostgreSQL e SeaweedFS para o fluxo completo de usuários e fotos. Não desabilite a validação de certificados globalmente.

## Ordem de execução

1. **Preparação > Obter CSRF**: guarda `csrfToken` e `csrfHeaderName`; o Cookie Jar armazena o cookie correspondente.
2. **Usuários > Cadastro > Criar usuário**: cria um usuário com username, e-mail e telefone únicos e endereço; guarda `loginEmail`, `userId` e `lastUser`.
3. **Usuários > Autenticação > Login**: envia e-mail ou username no campo `identifier`, junto com a senha. A resposta usa ApiResponse, com `data.expiresIn`; os JWTs chegam exclusivamente em Set-Cookie.
4. **Usuário autenticado**: consulta `GET /me` e atualiza os dados usados nas demais requisições.
5. **Renovar tokens**: envia o cookie de refresh e o header CSRF, sem body. O servidor invalida o refresh anterior e substitui ambos os cookies.
6. **Confirmar sessão renovada**: consulta `/me` com o novo cookie de acesso.
7. Execute consultas, atualizações e fotos. O upload de capa usa `POST /users/{userId}/photo?type=COVER` com o arquivo no campo multipart `photo`.
8. **Usuários > Perfil > Atualizar bio do perfil**: salva a bio em `PATCH /profiles/me`, preservando a capa. A primeira gravação cria o perfil.
9. **Consultar perfil autenticado**: confirma bio e `coverUrl`. Para baixar a capa, use **Fotos > Consultar URLs das fotos** e **Buscar foto da capa**.
10. **Logout**: encerra a sessão e limpa as variáveis do usuário na collection.

Para uma conta existente, execute Obter CSRF, defina `loginIdentifier` (e-mail ou username) e `loginPassword` no ambiente e execute Login. `loginIdentifier` preenchido no ambiente tem precedência. Quando estiver vazio, a requisição usa `loginEmail` do cadastro ou do ambiente. O campo JSON legado `email` continua aceito pela API.

O script da collection injeta o header CSRF automaticamente em POST, PUT, PATCH e DELETE. Se apagar os cookies ou reiniciar a sessão, execute Obter CSRF novamente. Os JWTs não são copiados para variáveis nem enviados em Authorization. O Bruno gerencia os cookies, seus caminhos e expirações; o refresh pode ser executado mesmo com o cookie de acesso vencido.

## Permissões atuais

O YAML libera cadastro, login, logout, CSRF, refresh e `GET /users/availability`. `/me`, PUT/PATCH de usuário, upload de fotos e GET/PATCH de `/profiles/me`, pesquisa em `/profiles` e consulta em `/profiles/username/{username}` exigem autenticação. As consultas de usuários e fotos aceitam USER ou ADMIN.

## Usuários e arquivos

O cadastro exige `username`, além de nome, e-mail, telefone e senha. Username e e-mail são únicos sem distinção de maiúsculas; telefone é único pela sequência de dígitos, ignorando a formatação. O cadastro aceita `addresses` com street, city, state, zipCode, country, referenceName, number e isDefault. As respostas incluem UUID e datas dos endereços, sem ID interno ou referência circular ao usuário. PUT e PATCH preservam os endereços existentes.

`fixtures/avatar.png` é usado nos uploads separados de perfil e capa. A capa enviada é persistida em `profile_entity.cover_url`, e a foto de perfil permanece no usuário. O cadastro gera username, e-mail e telefone únicos e usa a senha fictícia `loginPassword`. O fluxo grava dados reais no banco local. As variáveis de runtime não persistem após reiniciar o Bruno.

## Disponibilidade

As três requisições da pasta **Disponibilidade** usam `/users/availability?field=...&value=...`, com `field` igual a `username`, `email` ou `phone`. Configure `availabilityUsername`, `availabilityEmail` e `availabilityPhone` no ambiente para conferir valores livres ou já utilizados. A resposta mantém `data.available`.

As novas requisições são destinadas à execução e conferência manual, sem novos blocos de testes automatizados. As verificações antigas de cadastro e PUT foram removidas porque dependiam de dados fixos incompatíveis com os campos únicos.

## Pesquisa e visualização de perfis

Depois do login, use a pasta **Usuários > Perfil**:

1. **Pesquisar perfis por username**: configure `profileSearchQuery`, `profileSearchPage` e `profileSearchSize` no ambiente. A consulta usa `GET /profiles?query=luc&page=0&size=20`.
2. A busca atual considera somente trechos do username, sem diferenciar maiúsculas e minúsculas. `luc` pode encontrar `lucas123`; `lucs` não encontra `lucas` por aproximação. O nome de exibição ainda não participa da busca. `%` e `_` são literais. A estrutura aceita critérios adicionais no futuro.
3. A resposta paginada contém `data.content` com UUID, nome, username e URL da foto. O número da página começa em zero e o tamanho deve estar entre 1 e 100. O próprio usuário autenticado é excluído antes da paginação e da contagem. Não encontrar outros usuários retorna uma página vazia.
4. Copie um username retornado para `profileUsername`, sem `@`, e execute **Visualizar perfil por username**. A consulta usa `GET /profiles/username/{username}` e retorna bio, foto, capa e data de criação, além dos dados de apresentação. Um username inexistente retorna `404`.
5. Execute **Buscar foto do perfil visualizado** ou **Buscar capa do perfil visualizado** para obter a imagem binária pela API. Essas consultas usam `viewedUserId`, `viewedPhotoUrl` e `viewedCoverUrl`, preservando as variáveis do próprio usuário.

Essas consultas exigem autenticação, retornando `401` sem uma sessão válida. As novas requisições não contêm testes automatizados.

## Posts, mídia e feed

As pastas **Publicações** e **Feed** cobrem criação JSON, upload PNG/JPEG/GIF, incorporação do Tenor, consulta, edição, exclusão, curtidas, reposts e comentários.

- Execute Obter CSRF, Login e Usuário autenticado antes de usar as operações autenticadas.
- Criar publicação guarda `postId` e `lastPost`; para imagens locais também guarda `postMediaId`. Feed/listagens não selecionam automaticamente um post para edição ou exclusão. Para outro post, configure `postId` no ambiente com seu UUID público.
- Upload aceita até 4 imagens PNG/JPEG ou 1 GIF local, no máximo 10 MiB por arquivo; os exemplos usam `fixtures/avatar.png` e `fixtures/post-sample.gif`. Texto é opcional quando há mídia. O servidor verifica o conteúdo real e limita dimensões a 40 megapixels.
- O exemplo Tenor usa `tenorEmbed`, que aceita link ou HTML copiado do provedor. Não misture com `media`. A resposta tem `provider: TENOR` e `externalId`; o download autenticado é somente para anexos `UPLOAD`.
- `postAuthorId` vazio usa o usuário consultado por `/me`. `profilePostsTab` aceita `PUBLICATIONS`, `REPLIES` ou `LIKES`. O postId de um comentário fica em `commentId`, preservando o post pai selecionado.
- Editar altera apenas texto e exige autoria. Excluir remove texto e anexos, preservando respostas de outros autores. Execute exclusão manualmente em conteúdo descartável.
- O feed continua global cronológico, incluindo não amigos. `feedStrategy` permite selecionar explicitamente `chronological`. Páginas começam em zero; tamanhos entre 1 e 100. As listagens de posts/feed usam `hasNext`/`hasPrevious`, sem total geral.

## Notificações

**Listar notificações** retorna apenas a caixa da sessão e guarda o primeiro ID não lido em `notificationId` (ou o primeiro item). É possível selecionar outro ID no ambiente. A collection também permite contar não lidas, marcar uma ou todas como lidas.

Curtidas e reposts notificam somente quando a relação é criada; PUT repetido sem desfazer não duplica o aviso. Cada comentário novo notifica o autor do pai. Ações no próprio post não notificam. Desfazer uma interação preserva o histórico. A caixa do site é independente de envio por e-mail, que ainda não está configurado.

A leitura de um aviso de outro destinatário retorna 404. Post excluído preserva o aviso com `postAvailable: false`. Avisos de amizade possuem ator e `postId` nulo.

## Amigos e solicitações

A pasta **Amizades** inclui envio, consulta do estado, recebidas/enviadas, aceite, recusa, cancelamento, lista/contagem e desfazer amizade. `friendUserId` deve ser o UUID público da outra pessoa; vazio usa `viewedUserId` de Visualizar perfil por username.

Fluxo manual com duas contas:

1. Entre como A, selecione B em `friendUserId` e envie a solicitação. A verá `REQUEST_SENT`.
2. Faça logout e entre como B. Atualize `/me`, selecione o UUID de A no ambiente e consulte Recebidas ou a relação. B verá `REQUEST_RECEIVED`.
3. B aceita ou recusa. Aceitar muda o estado dos dois para `FRIENDS`, incrementa a contagem e notifica A.
4. Para cancelar, entre como quem enviou. Para desfazer amizade, qualquer uma das partes pode executar a ação depois do aceite.

Pedidos cruzados não viram amizade automaticamente; há uma linha única por par e bloqueio transacional das mudanças. Duplicatas não geram avisos extras. Solicitar a si mesmo retorna 400; ação reservada à outra parte retorna 403; transição incompatível retorna 409. O contador `friendCount` nos perfis inclui apenas amizades aceitas, inclusive no perfil visitado, usando o UUID do dono do perfil.

## Ambientes pessoais e segurança do repositório

Os ambientes versionados têm apenas exemplos. Defina contas reais pelo ambiente local do Bruno; `environments/*.local.bru` é ignorado pelo Git. Se existente, `Pessoal.local` preserva a configuração particular anterior e não é enviado ao repositório. O login agora resolve `loginIdentifier`/`loginEmail` para `loginValue`, sem usuário fixo no body.

As requisições novas têm scripts apenas para preparar e capturar variáveis de uso manual, sem blocos novos de testes automatizados. Não execute a collection inteira indiscriminadamente: cadastro, posts, exclusões e mudanças de amizade alteram dados. Antes de usar as novas rotas, reinicie a API atualizada para criar as tabelas e aplicar as migrações de desenvolvimento.
