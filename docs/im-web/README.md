# API

A API do Web Application é feita através de rotas do [NextJS](https://nextjs.org/). Através das páginas `api` no NextJS, o banco de dados do Im Connection é consultado através do [Prisma ORM](https://prisma.io).

### Keep Alive

A chamada de _KeepAlive_ é uma comunicação entre o `Plug` e a `API`. Isso é feito a cada "x segundos" e, assim, a `API` consegue reconhecer/reportar que um determinado `Plug` está operacional.

- `Plug` envia um payload de _Keep Alive_ para a `API`:

```ts
{
  "plug_code": 123, // Int
  "status_code": 1 // Int
}
```

- Recebe uma resposta da API "vazia" com HTTP Status 204 (No Content)

### Sessão de Carga via TAG

Quando o cliente deseja fazer uso de um `Plug` (ou seja, uma operação de carga), é necessário que a `API` seja informada dessa intenção. Para isso o fluxo é:

- Cliente "encosta" a `Tag` no `Plug`, e a leitura do RFID acontece.

- `Plug` inicia negociação com a `API`, enviando pedido de sessão de carga para a tag lida:

```ts
{
    "plug_code": 123, // Int
    "tag_code": 13918611076, // Int
}
```

- API valida a tag, encontra usuário, verifica cartão de crédito\* etc, e:

- Se as validações acima estiverem `OK`, a `API` responde com HTTP Status 200:

```ts
{
  "code": "12a462f1-8fbf-4e27-8012-57a38a0d68b5"
}
```

- Em caso de `ERRO`, API response com HTTP Status >= 400:

```js
{
  "error": true,
  "error_message": "tag não encontrada" // mensagem de erro pode variar conforme validação
}
```

#### Código Charge Session

Quando sessão de carga é criada, a `API` devolve um `code`, como pode ser visto acima. Este `code` é um [UUID Version 4](https://en.wikipedia.org/wiki/Universally_unique_identifier). Este UUID deverá ser "salvo" pelo `Plug` para ser reenviado futuramente **durante todos os [sinais de carga](/im-web/README?id=sinal-de-carga)**, até que a carga se encerre. Uma vez que o processo de carga seja **encerrado** e todos os sinais de carga tiverem sido **enviados**, esse `UUID` deve ser **descartado** pelo `Plug`.

### Sinal de Carga

Após uma [Sessão de Carga](im-web/README?id=sessão-de-carga-via-tag) ter sido criada, o `Plug` pode começar enviar à `API` informações sobre o andamento da carga a "cada x segundos". A isso dá-se o nome de Sinal de Carga:

```ts
{
    "charge_session_code": "12a462f1-8fbf-4e27-8012-57a38a0d68b5",
    "plug_code": 1,
    "tag_code": 13918611076,
    "total_time": 100,
    "charge_time": 20,
    "voltage": 120.00,
    "current": 15.00,
    "status_code": 1,
    "finished": false,
}
```

O payload acima pode ser detalhado da seguinte maneira:

- charge_session_code `(String)` Código da sessão de carga
- plug_code: `(Int)` ID do `Plug`
- tag_code: `(Int)` RFID da `Tag`
- total_time: `(Int)` Duração **total**, em segundos, que deve durar a sessão de carregamento
- charge_time: `(Int)` Tempo atual, em segundos, (contador) da sessão de carga em andamento
- voltage: `(Int)` Voltagem no momento do envio
- current: `(Int)` Corrente no momento do envio
- status_code: `(Int)` Status do `Plug`
- finished: `(Boolean)`

> **Atenção!** Se a flag `finished` for enviada como `true`, isso indica que o `Plug` deseja **encerrar a sessão de carga** na `API`. A partir deste momento, qualquer outro envio de Sinal de Carga com o mesmo `charge_session_code` irá disparar um **erro na `API`**.

#### Respostas da API ao Sinal de Carga

Em caso de **sucesso** no registro do sinal de carga acima, a `API` responde HTTP status 200 com:

```ts
{
  "keep_going": true ou false
}
```

> A flag `keep_going` acima indica para o `Plug` se o mesmo deve continuar o fluxo de carga ou se este deve interromper a carga. Essa "interrupção forçada" pela `API` pode ocorrer por diferentes razões, como por exemplo:

1. O cliente solicitou remotamente o encerramento da carga pelo app no celular.
2. O cliente - ou o titular da conta - removeu a `Tag` ou, até mesmo, sua conta.

Em caso de **erro**, a API responde com HTTP status >= 400 com:

```ts
{
  "error": true,
  "error_message": "Tag não encontrada" // Mensagem de erro pode variar de acordo com as validações
}
```

#### Liberação do `Plug` para uma nova sessão de carga:

Ao final do processo de carga - seja qual for o motivo - o `Plug` deve enviar uma mensagem [Keep Alive](/im-web/README?id=keep-alive), indicando que encontra-se novamente liberado para novas sessões de carga.

#### Sessão de Carga "aberta".

Em caso de qualquer problema com o `Plug` (conexão, falta de energia elétrica, etc) **durante uma sessão de carga**, fará com que o Sinal de Carga com a flag `finished: true` nunca chegue à `API`.

Isso acarretará em uma Sessão de Carga com status "em aberto". Para remediar esses casos, haverá um **cron job no back-end** para varrer estas sessões abertas com o objetivo de encerrá-las e então realizar o devido processo de cobrança dos valores/sinais registrados.
