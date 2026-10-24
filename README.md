# Bruno — inxbtwt-api

Abra esta pasta no Bruno e selecione o ambiente **Local**. A API usa `http://localhost:8080`; inicie também PostgreSQL e SeaweedFS.

## Organização

```text
Usuários
├── Cadastro
│   └── Criar usuário
├── Consultas
│   ├── Consultar usuário
│   └── Listar usuários
├── Atualizações
│   ├── Atualizar usuário (PUT)
│   ├── Atualizar usuário parcialmente (PATCH)
│   └── Confirmar alterações persistidas
└── Fotos
    ├── Enviar foto do usuário
    ├── Enviar capa do usuário
    ├── Consultar URLs das fotos
    ├── Buscar foto do perfil
    └── Buscar foto da capa
```

A ordem das pastas e requisições está definida nos arquivos `.bru`. Execute a collection inteira ou a pasta **Usuários** no Runner para percorrer o fluxo completo.

## Uso individual

- **Criar usuário** envia apenas o objeto JSON no body (`name`, `email`, `phone`, `password`); gera um e-mail único e guarda `userId` e `lastUser`. Não envia arquivos nem acessa o bucket.
- As consultas e alterações reutilizam `userId`. Para trabalhar com um usuário existente, defina essa variável no Bruno.
- **Enviar foto** e **Enviar capa** guardam `photoUrl`/`photoKey` e `coverUrl`/`coverKey`.
- **Consultar URLs** precisa apenas de `userId`; preenche as URLs atuais e limpa variáveis de fotos ausentes. Perfil e capa podem ser nulos antes dos uploads.
- As buscas das imagens usam essas URLs. Envie uma capa antes de buscar a imagem da capa caso ainda não exista.
- O PUT e o PATCH atualizam dados cadastrais; as fotos são alteradas pelos uploads.

As variáveis criadas por scripts são de runtime e não persistem após reiniciar o Bruno. Scripts de comparação de dados usam `lastUser`, preenchida pelo cadastro, PUT e PATCH.

## Verificação dos mapeamentos de usuário

O cadastro verifica nome, e-mail e telefone retornados, as datas geradas e a ausência de senha e campos internos na resposta. O PUT compara o identificador e a data de criação com o estado anterior; execute o cadastro antes dele para preencher `lastUser`. O PATCH verifica que os campos não enviados foram preservados.

O cadastro aceita uma lista opcional `addresses` e inclui um endereço de exemplo. A resposta retorna os endereços com `addressId`, campos de endereço e datas, sem o ID interno ou referência ao usuário. PUT e PATCH preservam os endereços existentes.

## Arquivos e execução

`fixtures/avatar.png` é a imagem de teste usada apenas nos endpoints separados de upload; seu caminho é relativo à raiz da collection e continua válido em todas as pastas. Para testar outra foto, altere o arquivo no Body da requisição.

```powershell
npx --yes @usebruno/cli@4.2.1 run --env Local --sandbox safe
```

O fluxo cria e atualiza um usuário real no banco local e envia imagens ao bucket `inxbtwt-files`. A senha é fictícia. Sucessos JSON usam `ApiResponse`; os downloads retornam a imagem em bytes.
