# 로컬에 Parse-Server, Parse-Dashboard 및 PostgreSQL 설치 및 테스트

작성자: [Jeff Gu Kang](https://github.com/JeffGuKang/)

이 튜토리얼은 NodeJS 환경을 기본으로 합니다.
AWS에서 RDS로 PostgreSQL을 지원하기 때문에 MongoDB 대신 PostgreSQL을 사용하겠습니다.

1. DB로 PostgreSQL 설치
2. Parse-Server용 PostgreSQL database 생성
3. Parse-Server 실행
4. Postman을 통해 동작 테스트
5. Parse-Dashboard를 통해 데이터베이스 확인

## PostgreSQL 설치

https://www.enterprisedb.com/downloads/postgres-postgresql-downloads

Window Tutorial: http://www.postgresqltutorial.com/install-postgresql/

1. 설치된 버전 확인

```sh
postgres -V
```

2. psql 콘솔로 진입

```sh
psql postgres
```

3. testdb 데이터베이스 생성

```sh
CREATE DATABASE testdb;
```

4. 확인 및 나오기

```sh
\list
\q
```

## Parse Server 실행

### Parse Server(파스 서버)란

파스 서버는 Node.js 환경에서 구동할 수 있는 오픈 소스로 만들어진 각 언어별 SDK를 제공하는 데이터베이스 플랫폼입니다. 기존에 서비스로 운영하던 파스 백엔드를 페이스북이 인수 후 오픈 소스로 공개하였습니다. 일반적으로 mongodb와 PostgreSQL을 데이터베이스로 사용합니다.
[홈페이지](https://docs.parseplatform.org/)

### 로컬 환경에 Parse Server 설치 및 실행

1. 미리 설정된 Parse-Server [다운로드](https://github.com/JeffGuKang/awseb-parse-server) (별풍 대신 스타를 줍시다.)
2. package.json 모듈들 인스톨

```sh
$ npm install
or
$ yarn
```

3. `config/default.json` 파일 설정

```json
{
  "appname": "YOUR_APP_NAME", // 파스 대쉬보드에 표시될 앱 이름
  "localDatabaseURI": "postgres://localhost:5432/testdb", // 로컬 데이터베이스 설정
  "cloud": "./cloud/main.js",
  "appId": "YOUR_PARSE_APP_ID", // 파스 서버에서 사용할 앱 아이디
  "masterKey": "YOUR_MASTER_KEY", //// 파스 서버에서 사용할 마스터키 (유출금지)
  "fileKey": "YOUR_FILE_KEY", // 파일 저장시 접미사로 사용되는 키
  "parseAPIPrefix": "/parse",
  "port": 1337,
  "s3key": "",
  "s3secret": "",
  "s3name": "",
  "s3region": "",
  "dashboard-id": "YOUR_DASHBOARD_ID", // 파스 대쉬보드
  "dashboard-password": "YOUR_DASHBOARD_PASSWORD",
  "dashboard-serverURL": "http://localhost:1337/parse"
}
```

4. 로컬에서 파스 서버 실행하기

```sh
$ npm run start:local

> NODE_ENV=local node index
parse-server running on port 1337
```

1337 포트에서 파스 서버가 실행되었습니다.

브라우져에서 localhost:1337 로 접속하면 public폴더에 미리 넣어 놓은 index.html 파일을 확인할 수 있습니다.

![index.html](./images/parse_start.png "index.html")


index.js 파일 확인하며 대략적인 파스 서버의 실행코드를 살펴봅시다.

```js
...
var app = express();

// mongodb://username:password@ggg.com:27017/dbname
var databaseURI = nconf.get('localDatabaseURI');
if (process.env.NODE_ENV !== 'local') {
  databaseURI = `postgres://${process.env.RDS_USERNAME}:${process.env.RDS_PASSWORD}@${process.env.RDS_HOSTNAME}:${process.env.RDS_PORT}/${process.env.RDS_DB_NAME}`;
}
// Specify the connection string for your mongodb database
// and the location to your Parse cloud code
var api = new ParseServer({
  databaseURI,
  cloud: nconf.get('cloud'),
  appId: nconf.get('appId'),
  masterKey: nconf.get('masterKey'),
  fileKey: nconf.get('fileKey'),
  serverURL: 'http://localhost:1337/parse',  // Don't forget to add this line to change port or url
  // enableAnonymousUsers: true,
  liveQuery: {
    // classNames: ['Alram'],
  },
  // filesAdapter: s3Adapter
});


// Serve static assets from the /public folder
app.use('/', express.static(path.join(__dirname, '/public')))

// if (process.env.NODE_ENV !== 'local') {
  const config = {
    "apps": [
      {
        "serverURL": nconf.get('dashboard-serverURL'),
        "appId": nconf.get('appId'),
        "masterKey": nconf.get('masterKey'),
        "appName": nconf.get('appname'),
        "supportedPushLocales": ["en", "kr"]
      },
    ],
    "users": [
      {
        "user": nconf.get('dashboard-userId'),
        "pass": nconf.get('dashboard-userPassword')
      },
    ]
  };
  var dashboard = new ParseDashboard(config, {allowInsecureHTTP: true}); //  only for test

  app.use('/dashboard', dashboard);
// }

// Serve the Parse API on the /parse URL prefix
app.use(nconf.get('parseAPIPrefix'), api);

// Hello world
// app.get('/', function(req, res) {
//   res.status(200).send('Init.');
// });

var httpServer = require('http').createServer(app); // Need for livequery
var port = nconf.get('port');
httpServer.listen(port, function() {
  console.log('parse-server running on port %s', port);
});

ParseServer.createLiveQueryServer(httpServer);
```

### Postman을 통한 Parse Server의 Cloud function 동작 테스트

index.js를 보면 파스서버를 초기화 할때 cloud라는 파라미터를 확인할 수 있습니다. 해당 파일에 선언되어진 특정 함수들을 외부에서 REST API 혹은 SDK를 통해 호출할 수 있도록 합니다.

index.js

```js
var api = new ParseServer({
  databaseURI,
  cloud: nconf.get('cloud'),
  appId: nconf.get('appId'),
  masterKey: nconf.get('masterKey'),
  fileKey: nconf.get('fileKey'),
  serverURL: 'http://localhost:1337/parse',  // Don't forget to add this line to change port or url
  // enableAnonymousUsers: true,
  liveQuery: {
    // classNames: ['Alram'],
  },
  // filesAdapter: s3Adapter
});
```

cloude/main.js

```js
Parse.Cloud.define('hello', function(req, res) {
  return 'Hi';
});
```

main.js에 `Parse.Cloud.define`을 통해 선언되어진 `hello` 를 외부에서 호출하면 응답을 받을 수 있습니다.

`localhost:1337/parse/functions/hello` 이 주소를 통해 호출 가능합니다.

![cloud function-hello](./images/cloudfunction.png "cloud function-hello")

미리 Parse Server에 등록해놓은 AppId키 값을 헤더값으로 넣지 않으면 위와 같이 인증되지 않았다는 응답을 받게됩니다.
여기서는 [Postman](https://www.getpostman.com/)을 통해 테스트해 보도록 하겠습니다.

1. Postman 다운로드 및 실행
2. 새로운 요청 생성

- 방식 `POST`
- 주소: `localhost:1337/parse/functions/hello`
- Send 버튼을 통해 테스트해봅니다. 아까와 같은 인증이 필요하다는 응답이 나옵니다.
- Headers에 값 입력: X-Parse-Application-Id에는 미리 설정한 Parse App ID를, Content-Type에는 application/json을 입력합니다.
- Send 버튼으로 다시 테스트 했을때 응답으로 "result": "Hi" 가 나타나면 성공입니다. Postman에서 만든 요청은 저장을 해주세요.

![Postman](./images/postman.png "Postman")

3. 이번에는 기존 코드를 응용하여 사용자 입력을 받아 정해진 응답을 주는 API를 만들어 보겠습니다.

cloud/main.js 하단에 다음 코드를 추가합니다.

```js
Parse.Cloud.define('whoami', function(req, res) {
  const name = req.params.name;
  const money = req.params.money;

  const result = `Hello. ${name}! You will earn $${money}!!!`
  return result;
});
```

name과 money를 파라미터로 받아 메세지를 출력해주는 API입니다.

기존 Postman에 만든 요청을 복제하여 새로운 요청을 만듭니다.

![postman_duplication](./images/postman_duplication.png "postman_duplication")

별다른 추가 작업 없이 Send를 통해 API를 호출해봅시다.

```json
{
    "code": 141,
    "error": "Invalid function: \"whoami\""
}
```

해당 API가 존재하지 않는다는 오류코드가 나옵니다. 코드를 변경 후 NodeJS 서버를 재실행해야 적용이 되기 때문입니다.
`npm run start:local`을 통해 서버를 실행하고 있는 콘솔에서도 에러 메세지를 확인할 수 있습니다.

서버를 실행하고 있는 콘솔에서 파스 서버를 ctl+c를 통해 중단시킨 후 재실행해봅니다.

다시 Send를 통해 API를 호출하면

```json
{
    "result": "Hello. undefined! You will earn $undefined!!!"
}
```

위와 같은 결과가 나옵니다. 아직 whoami API에서 사용하는 name, money 인자값을 전달하지 않아 undefined가 텍스트로 출력되고 있습니다.
아까 cloud/main.js에 추가한 코드에서 `req.params.name` 이 이름을 받아오는 코드입니다. `req.params`에 입력한 데이터들이 전달되어 코드상에서 컨트롤할 수 있습니다.

Postman에서 Body 부분에 name, money 값을 입력하고 위 과정을 반복해보면 아래와 같이 정상적인 결과를 얻을 수 있습니다.

![postman_post](./images/postman_post.png "postman_post")

차후 데이터베이스의 값을 다루는 API를 추가하게 되면 게임에서 유저들의 순위를 변동한다던가, 캐쉬를 적립한다거나 하는 등의 다양한 동작을 다룰 수 있습니다.

## Parse Dashboard를 통한 관리자 모드 접속

지금 사용하고 있는 프로젝트에는 Parse Dashboard라는 파스 서버의 데이터베이스 관리 앱에 관한 설정이 추가되어 있습니다.

index.js

```js
const config = {
  "apps": [
    {
      "serverURL": nconf.get('dashboard-serverURL'),
      "appId": nconf.get('appId'),
      "masterKey": nconf.get('masterKey'),
      "appName": nconf.get('appname'),
      "supportedPushLocales": ["en", "kr"]
    },
  ],
  "users": [
    {
      "user": nconf.get('dashboard-userId'),
      "pass": nconf.get('dashboard-userPassword')
    },
  ]
};
var dashboard = new ParseDashboard(config, {allowInsecureHTTP: true}); //  only for test

app.use('/dashboard', dashboard);
```

appname, dashboard-userId와 패스워드 부분을 입맛에 맞게 편집해주시면 됩니다.

config/default.json

```json
{
  "appname": "YOUR_APP_NAME",
  "localDatabaseURI": "postgres://localhost:5432/testdb",
  "cloud": "./cloud/main.js",
  "appId": "YOUR_PARSE_APP_ID",
  "masterKey": "YOUR_MASTER_KEY",
  "fileKey": "YOUR_FILE_KEY",
  "parseAPIPrefix": "/parse",
  "port": 1337,
  "s3key": "",
  "s3secret": "",
  "s3name": "",
  "s3region": "",
  "dashboard-userId": "YOUR_DASHBOARD_ID",
  "dashboard-userPassword": "YOUR_DASHBOARD_PASSWORD",
  "dashboard-serverURL": "http://localhost:1337/parse"
}
```

그 후 다시 파스 서버를 재시작하고 http://localhost:1337/dashboard 주소로 접속을 시도합니다.

로그인 화면에 위에서 설정한 대쉬보드의 id, password를 입력합니다.

![dashboard_login](./images/dashboard_login.png "dashboard_login")

드디어 대쉬보드의 메인 화면을 확인할 수 있습니다. 들어가서 이것저것 구경해보세요.

![dashboard_main](./images/dashboard_main.png "dashboard_main")

### CLI을 통한 대쉬보드 접속

프로젝트에는 미리 정의된 `parse-dashboard-config-local.json` 파일이 있습니다.

```json
{
  "apps": [
    {
      "serverURL": "http://localhost:1337/parse/",
      "appId": "YOUR_PARSE_APP_ID",
      "masterKey": "YOUR_MASTER_KEY",
      "appName": "YOUR_APP_NAME",
      "supportedPushLocales": ["en", "kr"]
    }
  ]
}
```

index.js에서 설정했던 것과 유사한 설정입니다. 다만 local에서 실행하므로 masterKey를 입력하여 따로 아이디와 비밀번호 없이 해당 파스 서버에 접속할 수 있습니다. 기본 포트는 4040을 사용합니다.

```
$ parse-dashboard --config parse-dashboard-config-local.json
```

```
> parse-dashboard --config parse-dashboard-config-local.json

The dashboard is now available at http://0.0.0.0:4040/
```

위 명령어는 `package.json`에 미리 정의를 해 놓았으므로 `npm run dashboard:local`로도 실행할 수 있습니다.
http://0.0.0.0:4040 혹은 http://localshot:4040 를 통해 로그인 과정 없이 접속이 되는것을 확인하세요.

## Conclusion

- 오픈소스 데이터베이스인 PostgreSQL을 설치하고 프로젝트에서 사용할 DATABASE를 생성하였습니다.
- 로컬 NodeJS 환경에서 express 프레임워크를 통해 Parse Server를 띄워 보았습니다.
- Parse Server의 클라우드 코드를 통해 API를 생성하고 Postman을 통해 테스트해 보았습니다.
- Parse Dashboard로 Parse Server 데이터베이스에 접속하여 관리자 페이지를 동작하였습니다.

다음은 위에서 구성한 Parse Server를 실제로 AWS 에 배포하여 구동해보도록 하겠습니다.

[이전 과정으로](./README.md) / [다음 과정으로](./Tutorial2.md)