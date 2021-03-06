# Configurando o *MongoDB* como banco de dados
 
 /todo

## Introdução ao MongoDB

/todo

## Configurando o banco de dados com *Mongoose*

Para integrar nossa aplicação com o *MongoDB* iremos utilizar o [*Mongoose*]() que é *ODM (Object Document Mapper)*. O *Mongoose* irá abstrair o acesso ao banco de dados e ainda irá se responsabilizar por transformar os dados do banco em *Models*, o que irá facilitar a estruturação de nossa aplicação com o padrão *MVC*.

Para instalar o *Mongoose* basta executar o seguinte comando *npm*:

```sh
$ npm install mongoose --save
```

Após isso o *Mongoose* estará disponível para ser utilizado. O próximo passo será configurar a aplicação para conectar com o banco de dados, para isso crie um diretório chamado *config* dentro de *src* e dentro dele um arquivo chamado ***database.js*** que será responsável por toda configuração do banco de dados.
A estrutura de diretórios deve estar assim:

```
├── src
│   ├── app.js
│   ├── config
│   │   └── database.js
│   ├── controllers
│   │   └── products.js
│   └── routes
│       ├── index.js
│       └── products.js
```

A primeira coisa que deve ser feita no *database.js* é importar o módulo do *Mongoose* como no código abaixo:

```javascript
import mongoose from 'mongoose';
```

O próximo passo é informar qual biblioteca de *promises* está sendo usada, isso é uma necessidade da versão 4 do *Mongoose*. Como estamos usando o *Node.js* na versão 6 e ele já possui *promises* por padrão será necessário fazer somente o seguinte:

```javascript
mongoose.Promise = Promise;
```

Aqui é dito para o *Mongoose* utilizar a *promise* oficial do *Node.js*. Seguindo a configuração do banco de dados é necessário informar  a *url* onde está o *MongoDB*, no meu caso está no meu computador então será *"localhost"*  seguido do nome que daremos ao banco de dados:

```javascript
const mongodbUrl = process.env.MONGODB_URL || 'mongodb://localhost/test';
```

Note que primeiro é verificado se não existe uma variável de ambiente, caso não exista é usado o valor padrão que ira se referir ao *"localhost"* e ao banco de dados *"test"*. Dessa maneira poderemos utilizar o *MongoDB* tanto para testes quanto para rodar o banco da aplicação de verdade sem precisar alterar o código.

No passo seguinte iremos criar uma função para conectar no banco de dados.

```javascript
const connect = () => mongoose.connect(mongodbUrl);
```

Aqui é criado uma função que retorna uma conexão com o *MongoDB*, esse retorno é uma *promise*, ou seja, somente quando a conexão for estabelecida a *promise* será resolvida, isso é importante pois precisamos garantir que nossa aplicação só vai estar disponível depois que o banco de dados estiver conectado e acessivel.

O último passo é exportar o módulo de configuração do banco de dados:

```javascript
export default {
  connect
}
```

O código do *"database.js"* deve estar similar a este:

```javascript
import mongoose from 'mongoose';

mongoose.Promise = Promise;

const mongodbUrl = process.env.MONGODB_URL || 'mongodb://localhost/test';

const connect = () => mongoose.connect(mongodbUrl);

export default {
  connect
}
```

Pronto, o banco de dados está configurado, o próximo passo será integrar com a aplicação para que ela inicialize o banco sempre que for iniciada. 

## Integrando o *Mongoose* com a aplicação

O módulo responsável por inicializar a aplicação é o *app*, então será ele que irá garantir que o banco estará disponível para que a aplicação possa consumi-lo.
Vamos alterar o ***"app.js"*** para que ele integre com o banco de dados. Atualmente ele está assim:

```javascript
import express from 'express';
import bodyParser from 'body-parser';
import routes from './routes';

const app = express();
app.use(bodyParser.json());
app.use('/', routes);

export default app;
```

A primeira coisa a fazer é importar o módulo responsável pelo banco de dados, o *"database.js"** que fica dentro do diretório *config*. Os *imports* devem ficar assim:

```diff
 import express from 'express';
 import bodyParser from 'body-parser';
 import routes from './routes';
+ import database from './config/database';
```

A seguir muramores um pouco o código anterior que utiliza o *express* e as rotas movendo a seguinte parte:

```diff
- app.use(bodyParser.json());
- app.use('/', routes);
 
- export default app;
```

As partes em vermelho serão movidas para dentro de uma nova função, como no código abaixo:

```diff
+ const configureExpress = () => {
+  app.use(bodyParser.json());
+  app.use('/', routes);
+
+  return app;
+};
```

Aqui foi criada uma função nomeada como ***configureExpress*** que tera a responsabilidade de configurar o *express* e retornar uma nova instância de aplicação configurada.

A última etapa da nossa alteração é inicializar o banco antes da aplicação. Como o *moongose* retorna uma *promise*, iremos esperar ela ser resolvida para então chamar a função que configura o *express* que criamos anteriormente.

```diff
+ export default () => database.connect().then(configureExpress);
```

Nesse bloco exportamos uma função que retorna uma *promise*. A primeira coisa chamada é a função *"connect"* do *database* que criamos na etapa anterior, assim que essa *promise* for resolvida, o que significa o banco de dados estar disponível, então é chamada a função *configureExpress* que irá configurar o *express* e retornar uma nova instância da aplicação. Esse *pattern* é conhecido como *promises* encadeadas, ou em ingles, *chained promises*.

Note que a função *"configureExpress"* não precisaria existir, poderíamos ter uma função diretamente dentro do *"then"*  do *"connect"* e nela configurar o *express*, mas criado uma função que descreva o que está sendo feito torna o código mais intuitivo e desacoplado, pode se ler mais sobre o assunto [nesta *issue* do *airbnb*](https://github.com/airbnb/javascript/issues/216).

O *"app.js"* depois de alterado deve estar assim:

```javascript
import express from 'express';
import bodyParser from 'body-parser';
import routes from './routes';
import database from './config/database'

const app = express();

const configureExpress = () => {
  app.use(bodyParser.json());
  app.use('/', routes);

  return app;
};

export default () => database.connect().then(configureExpress);
```

Como alteramos o *app* para retornar uma função que retorna uma *promise*, será necessário alterar o *"server.js"* para fazer a inicialização de maneira correta.


## Alterando a inicialização

O *"server.js"* é o arquivo responsável por inicializar a aplicação, chamando o *app*. Como alteramos algumas coisas na etapa anterior precisamos atualizar ele.
Vamos começar alterando o nome do módulo na importação:

```diff
- import app from './src/app';
+ import setupApp from './src/app';
```

O módulo foi alterado de *app* para *setupApp*, por que? Porque agora ele é uma função e esse nome reflete mais a sua responsabilidade.
O próximo passo é alterar a maneira de como o *app* é chamado:

```diff
-app.listen(port, () => {
-  console.log(`app running on port ${port}`);
-});
+setupApp()
+ .then(app => app.listen(port, () => console.log(`app running on port ${port}`)))
+ .catch(error => {
+   console.error(error);
+   process.exit(1);
+ });
```

Como o código anterior devolvia uma instância da aplicação diretamente, era apenas necessário chamar o método *"listen"* do *express* para inicializar a aplicação. Como agora temos uma função que retorna uma *promise* devemos chama-la, ela vai inicializar o *app*, inicializando o banco e configurando o *express* e retornando uma nova instância da aplicação, ai então será possível inicializar a aplicação chamando o *"listen"*.

Até esse momento espero que já tenham lido a especificação de *promises* mais de 10 vezes e sejam mestres na implementação. Quando um problema ocorre a *promise* é rejeitada, e esse erro pode ser tratado usando um *catch* como no código acima.
Acima recebemos o erro e mostramos ele em um *"console.log"*, e logo encerramos o processo do *Node.js* com o código 1 (falso).
Dessa maneira o processo é finalizado informando que houve um erro em sua inicialização. Informar o código de saída é uma boa prática, esse padrão de finalizar o processo com código de erro é conhecido como *"graceful shutdown"* e faz parte da lista do [*12 factor app*](https://12factor.net/) de boas práticas para desenvolvimento de *software* moderno.

As alterações necessários para integrar com o banco de dados estão finalizadas, basta executar os testes de integração para garantir:

```shell
$ npm run test:integration
```

A saida será:

```shell
 Routes: Products
    GET /products
      1) should return a list of products


  0 passing (152ms)
  1 failing

  1) Routes: Products GET /products should return a list of products:
     TypeError: Cannot read property 'get' of undefined
      at Context.done (test/integration/routes/products_spec.js:21:7)
```

O teste quebrou! Calma, isso era esperado.
Assim como o *"server.js"* o teste de integração inicia a aplicação usando o módulo *app*, então ele também deve ser alterado para lidar com a *promise*.

Vamos começar alterando o ***"helpers.js"*** dos testes de integração, como no código abaixo:

```diff
-import app from '../../src/app.js';
+import setupApp from '../../src/app.js';
  
-global.app = app;
-global.request = supertest(app);
+global.setupApp = setupApp;
+global.supertest = supertest;
```

Aqui, como no *"server.js"*, alteramos o nome do módulo de *app* para *setupApp* e o exportamos globalmente. Também removemos o *request* do conceito global que era uma instância do *supertest* com o *app* configurado, deixaremos para fazer isso no próximo passo.

Agora é necessário alterar o ***"products_spec.js"*** para inicializar a aplicação antes de começar a executar os casos de teste usando o *callback* *"before"* do *Mocha*:

```diff
describe('Routes: Products', () => {
+  let request;
+
+  before(()=> {
+    return setupApp()
+      .then(app => {
+        request = supertest(app)
+      })
+  });
+
```

No bloco acima é criado um *let* para o *"request"* do *supertest* e no *"before"*  a aplicação é inicializada, assim que o *"setupApp"* retornar uma instância da aplicação é possível inicializar o *supertest* e atribuir a *let "request"* que definimos anteriormente.

Executando os testes novamente, a saída deve ser a seguinte:

```shell
  Routes: Products
    GET /products
      ✓ should return a list of products


  1 passing (336ms)
```

Caso receba um erro como esse *"MongoError: failed to connect to server [localhost:27017] on first connect"*:

```shell
  Routes: Products
    1) "before all" hook


  0 passing (168ms)
  1 failing

  1) Routes: Products "before all" hook:
     MongoError: failed to connect to server [localhost:27017] on first connect
      at Pool.<anonymous> (node_modules/mongodb-core/lib/topologies/server.js:326:35)
      at Connection.<anonymous> (node_modules/mongodb-core/lib/connection/pool.js:270:12)
      at Socket.<anonymous> (node_modules/mongodb-core/lib/connection/connection.js:175:49)
      at emitErrorNT (net.js:1272:8)
      at _combinedTickCallback (internal/process/next_tick.js:74:11)
      at process._tickCallback (internal/process/next_tick.js:98:9)
```

Significa que o *MongoDB* não está executando em *localhost* na porta 7000, verifique ele e tente novamente.

O código desta etapa esta disponivel [aqui](https://github.com/waldemarnt/building-testable-apis-with-nodejs-code/tree/step6).
