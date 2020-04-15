# Dockerizando um app Node.js e MongoDB

Este tutorial mostrará como configurar um ambiente de desenvolvimento para um aplicativo Node.js usando Docker. Serão criados dois contêineres - um para o aplicativo Node e outro para o banco de dados MongoDB - com o Docker Compose. Como este aplicativo funciona com o Node e o MongoDB, nossa configuração fará o seguinte:

- Sincronizar o código do aplicativo no host com o código no contêiner para facilitar as alterações durante o desenvolvimento.
- Garante que as alterações no código do aplicativo funcionem sem um reinício.
- Cria um usuário e um banco de dados protegido por senha para os dados do aplicativo.
- Persistir esses dados.

No final deste tutorial, teremos um aplicativo funcional de informações sobre tubarões sendo executado em contêineres do Docker. 

--------------------------------------------------------------------------

## Pré-requisitos
Para seguir este tutorial, será necessário:

- Um servidor de desenvolvimento executando o Ubuntu 18.04, junto com um usuário não raiz com privilégios sudo e um firewall ativo. 
- O Docker instalado no seu servidor.
- O Docker Compose instalado no seu servidor.

## Passo 1: Clonando o projeto e modificando as dependências
Clone o repositório em um diretório chamado **node_project**:
 ```
$ git clone https://github.com/do-community/nodejs-mongo-mongoose.git node_project
 ```

 Navegue até o diretório **node_project**:
```
$ cd node_project
```

Abra o arquivo do projeto **package.json** e, por baixo das dependências do projeto e acima da chave de fechamento, crie um novo objeto **devDependencies** que inclua o **nodemon** *(ao executar o aplicativo com o nodemon, fica garantido que ele será reiniciado automaticamente sempre que você fizer alterações no seu código)*.
#### /node_project/package.json
 ```js
...
//"dependencies": {
//    "ejs": "^2.6.1",
//    "express": "^4.16.4",
//    "mongoose": "^5.4.10"
//  },
  "devDependencies": {
    "nodemon": "^1.18.10"
  }   
//}
 ``` 

Salve e feche o arquivo quando terminar a edição.

Com o código do projeto funcionando e suas dependências modificadas, você pode seguir para a refatoração do código para um fluxo de trabalho em contêiner. 

## Passo 2: Configurando o aplicativo para trabalhar com contêineres
Modificar o aplicativo para um fluxo de trabaho em contêiner significa tornar nosso código mais modular. Os contêineres oferecem portabilidade entre ambientes, e nosso código deve refetir isso mantendo-se dissociado do sistema operacional subjacente o máximo possível. Para isso, vamos refatorar o código para fazer uso da propriedade **process.env**, que retorna um objeto com informações sobre seu ambiente de usuário em tempo de execução. Podemos usar este objeto no código para atribuir dinamicamente informações de configuração em tempo de execução com variáveis de ambiente. 

Vamos começar com o **app.js**, nosso principal ponto de entrada do aplicativo. Dentro dele, você verá uma definição constante para uma **port**, bem como uma função **listen** que usa essa constante para especificar a porta na qual o aplicativo irá escutar. 
#### node_project/app.js
```javascript
...
const port = 8080;
...
app.listen(port, function () {
  console.log('Example app listening on port 8080!');
});
```
Vamos redefinir a constante **port** para permitir uma atribuição dinâmica em tempo de execução usando o objeto **process.env**. Faça as alterações a seguir na definição da constante e função **listen**:
#### node_project/app.js
```js
...
const port = process.env.PORT || 8080;
...
//app.listen(port, function () {
  console.log(`Example app listening on ${port}!`);
//});
```
Essa nova definição da constante atribui a porta dinamicamente usando o valor passado em tempo de execução ou 8080. 

Em seguida, vamos modificar a informação de conexão de banco de dados para remover quaisquer credenciais de configuração. Abra o arquivo **db.js**, que contém essa informação.

Atualmente, o arquivo faz o seguinte:
- Importa o Mongoose.
- Define as credenciais do banco como constantes, incluindo o nome de usuário e a senha.
- Conecta-se ao banco de dados utilizando o método **mongoose.connect**.

Nosso primeiro passo na modificação do arquivo será redefinir as constantes que incluem informações sensíveis. Atualmente, elas se parecem com isso:
#### node_project/db.js
```js
...,
const MONGO_USERNAME = 'sammy';
const MONGO_PASSWORD = 'your_password';
const MONGO_HOSTNAME = '127.0.0.1';
const MONGO_PORT = '27017';
const MONGO_DB = 'sharkinfo';
...
```
Em vez de codificar essas informações de forma rígida, é possível usar o objeto **process.env** para capturar os valores de tempo de execução para essas constantes. Modifique o bloco para que se pareça com isso:
#### node_project/db.js
```js
...,
const {
  MONGO_USERNAME,
  MONGO_PASSWORD,
  MONGO_HOSTNAME,
  MONGO_PORT,
  MONGO_DB
} = process.env;
...
```

Nesse ponto, você modificou o **db.js** para trabalhar com as variáveis de ambiente do seu aplicativo, mas ainda precisa de uma maneira de passar essas variáveis ao aplicativo. Vamos criar um arquivo **.env** com valores que você pode passar ao aplicativo em tempo de execução. 

Este arquivo incluirá as informações que você removeu do **db.js**: o nome de usuário e senha para o banco de dados, além da configuração de porta e nome do banco. 
#### node_project/.env
```
MONGO_USERNAME=<seu_usuario>
MONGO_PASSWORD=<sua_senha>
MONGO_PORT=27017
MONGO_DB=<nome_do_banco>
```
Note que removemos a configuação de host que originalmente apareceu em **db.js**. Agora, vamos definir nosso host no nível do arquivo do Docker Compose, junto com outras informações sobre os serviços e contêineres. 

Como seu arquivo **.env** contém informações sensíveis, você vai querer que ele esteja incluído nos arquivos **.dockerignore** e **.gitignore** do seu projeto para que ele não copie para o seu controle de versão ou contêineres. 

Abra seu arquivo **.dockerignore**:
#### node_project/.dockerignore
```
...
.gitignore
.env
```
O arquivo **.gitignore** neste repositório já inclui o **.env**, mas sinta-se à vontade para verificar se ele está lá.

Neste ponto, você extraiu informações sensíveis do seu código de projeto com sucesso e tomou medidas para controlar como e onde essas informações são copiadas. Agora, você pode adicionar mais robustez ao seu código de conexão de banco de dados para otimizá-lo para um fluxo de trabalho em contêiner.

## Passo 3: Modificando as configurações de conexão de banco de dados
Nosso próximo passo será tornar nosso método de conexão do banco de dados mais robusto adicionando códigos que lidem com casos onde o aplicativo falhe em se conectar ao banco. Introduzir este nível de resistência ao código é uma prática recomendada ao trabalhar com contêineres usando o Compose.

Abra o **db.js** para edição. 

Atualmente, o método **connect** aceita uma opção que diz ao Mongoose para usar o novo analisador de URL do Mongo. Vamos adicionar mais algumas opções a este método para definir parâmetros para tentativas de reconexão. Podemos fazer isso criando uma constante **options** que inclua as informações relevantes, além da nova opção de analisador de URL. Abaixo das suas constantes do mongo, adicione a seguinte definição para uma constante option:
#### node_project/db.js
```js
...
//const {
//  MONGO_USERNAME,
//  MONGO_PASSWORD,
//  MONGO_HOSTNAME,
//  MONGO_PORT,
//  MONGO_DB
//} = process.env;

const options = {
  useNewUrlParser: true,
  reconnectTries: Number.MAX_VALUE,
  reconnectInterval: 500,
  connectTimeoutMS: 10000,
};
...
```
A opção **reconnectTries** diz ao Mongoose para continuar tentando se conectar indefinidamente, ao mesmo tempo que a **reconnectInterval** define o período entre tentativas de conexão em milissegundos. A **connectTimeoutMS** define 10 segundos como o período que o condutor do Mongo irá esperar antes de falhar a tentativa de conexão. 

Agora, podemos usar as novas constantes **options** no método **connect** do Mongoose para ajustar nossas configurações de conexão. Também devemos adicionar uma promise paralidar com possíveis erros de conexão. 

Atualmente, o método **connect** se parece com isso:
#### node_project/db.js
```js
...
mongoose.connect(url, {useNewUrlParser: true});
```
Substitua o método **connect** pelo seguinte código, que inclui as constantes **options** e uma promise:
#### node_project/db.js
```js
...
mongoose.connect(url, options).then( function() {
  console.log('MongoDB is connected');
})
  .catch( function(err) {
  console.log(err);
});
```
Agora, você adicionou resiliência ao código do seu aplicativo para lidar com casos onde ele talvez falhasse em se conectar ao seu banco de dados. Com esse código funcionando, você pode seguir em frente para definir seus serviços com o Compose.

## Passo 4: Definindo serviços com o Docker Compose
Com seu código refatorado, você está pronto para escrever o arquivo **docker-compose.yml** com as definições do serviço. Um serviço no Compose é um contêiner em execução e as definições de serviço — que você incluirá no seu arquivo **docker-compose.yml** — contém informações sobre como cada imagem de contêiner será executada. A ferramenta Compose permite que você defina vários serviços para construir aplicativos multi-contêiner.

No entanto, antes de definir nossos serviços, vamos adicionar uma ferramenta ao nosso projeto chamada **wait-for** para garantir que nosso aplicativo tente se conectar apenas ao nosso banco de dados assim que as tarefas de inicialização do banco de dados estiverem completas. Este script de empacotamento usa o **netcat** para verificar se um host e porta específicos estão ou não aceitando conexões TCP. Usar ele permite que você controle as tentativas do seu aplicativo para se conectar ao seu banco de dados testando se ele está ou não pronto para aceitar conexões.

Queremos que nosso aplicativo se conecte apenas quando as tarefas de inicialização do banco de dados, incluindo a adição de um usuário e senha ao banco de dados de autenticação do admin, estejam completas.

Crie um arquivo chamado **wait-for.sh** e cole o códio a seguir para criar a função de votação:
#### node_project/wait-for.sh
```sh
#!/bin/sh

# original script: https://github.com/eficode/wait-for/blob/master/wait-for

TIMEOUT=15
QUIET=0

echoerr() {
  if [ "$QUIET" -ne 1 ]; then printf "%s\n" "$*" 1>&2; fi
}

usage() {
  exitcode="$1"
  cat << USAGE >&2
Usage:
  $cmdname host:port [-t timeout] [-- command args]
  -q | --quiet                        Do not output any status messages
  -t TIMEOUT | --timeout=timeout      Timeout in seconds, zero for no timeout
  -- COMMAND ARGS                     Execute command with args after the test finishes
USAGE
  exit "$exitcode"
}

wait_for() {
  for i in `seq $TIMEOUT` ; do
    nc -z "$HOST" "$PORT" > /dev/null 2>&1

    result=$?
    if [ $result -eq 0 ] ; then
      if [ $# -gt 0 ] ; then
        exec "$@"
      fi
      exit 0
    fi
    sleep 1
  done
  echo "Operation timed out" >&2
  exit 1
}

while [ $# -gt 0 ]
do
  case "$1" in
    *:* )
    HOST=$(printf "%s\n" "$1"| cut -d : -f 1)
    PORT=$(printf "%s\n" "$1"| cut -d : -f 2)
    shift 1
    ;;
    -q | --quiet)
    QUIET=1
    shift 1
    ;;
    -t)
    TIMEOUT="$2"
    if [ "$TIMEOUT" = "" ]; then break; fi
    shift 2
    ;;
    --timeout=*)
    TIMEOUT="${1#*=}"
    shift 1
    ;;
    --)
    shift
    break
    ;;
    --help)
    usage 0
    ;;
    *)
    echoerr "Unknown argument: $1"
    usage 1
    ;;
  esac
done

if [ "$HOST" = "" -o "$PORT" = "" ]; then
  echoerr "Error: you need to provide a host and port to test."
  usage 2
fi

wait_for "$@"
```
Crie o executável do script:
```
$ chmod +x wait-for.sh
```
Em seguida, crie o arquivo **docker-compose.yml**. 
#### node_project/docker-compose.yml
```yml
version: '3'

services:
  nodejs:
    build:
      context: .
      dockerfile: Dockerfile
    image: nodejs
    container_name: nodejs
    restart: unless-stopped
    env_file: .env
    environment:
      - MONGO_USERNAME=$MONGO_USERNAME
      - MONGO_PASSWORD=$MONGO_PASSWORD
      - MONGO_HOSTNAME=db
      - MONGO_PORT=$MONGO_PORT
      - MONGO_DB=$MONGO_DB
    ports:
      - "80:8080"
    volumes:
      - .:/home/node/app
      - node_modules:/home/node/app/node_modules
    networks:
      - app-network
    command: ./wait-for.sh db:27017 -- /home/node/app/node_modules/.bin/nodemon app.js

  db:
    image: mongo:4.1.8-xenial
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MONGO_INITDB_ROOT_USERNAME=$MONGO_USERNAME
      - MONGO_INITDB_ROOT_PASSWORD=$MONGO_PASSWORD
    volumes: 
      - dbdata:/data/db
    networks:
      - app-network 

networks:
  app-network:
    driver: bridge

volumes:
  dbdata:
  node_modules: 
```

Com as definições do seu serviço instaladas, você está pronto para iniciar o aplicativo.

## Passo 5: Testando o aplicativo
Com seu arquivo **docker-compose.yml** funcionando, você pode criar seus serviços com o comando **docker-compose up**.

Também podemos testar se os dados inseridos persistirão caso você remova seu contêiner de banco de dados: **docker-compose down**.

Recrie os contêineres: **docker-compose up -d**.


