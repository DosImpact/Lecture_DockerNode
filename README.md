# 🛠 Docker + Node.js + nginx + Express > 로드벨런싱 + 멀티프로세싱 서버

# Install

```
npm install
```

# 특징

- 1. 서버로 데이터를 요청한다. IP와 지정된 하나의 포트로 > localhost:4000
- 2. nginx 가 알아서 여러개의 포트로 로드 벨런싱을 해준다.
     > localhost:4000, localhost:4001, localhost:4002 ...
- 3. 해당 포트와 연결된 도커 컨테이너 안의 서버와 연결시켜준다.
     > localhost:4000 > 3000: docker container 1
     > localhost:4001 > 3000: docker container 2
     > localhost:4001 > 3000: docker container 3
- 4. 끝!

# 도커 파일

```python
FROM node:carbon
MAINTAINER Doyoung ypd03008@gmail.com

#app 폴더 만들기 - NodeJS 어플리케이션 폴더
RUN mkdir -p /app
#winston 등을 사용할떄엔 log 폴더도 생성

#어플리케이션 폴더를 Workdir로 지정 - 서버가동용
WORKDIR /app

#서버 파일 복사 ADD [어플리케이션파일 위치] [컨테이너내부의 어플리케이션 파일위치]
#저는 Dockerfile과 서버파일이 같은위치에 있어서 ./입니다
ADD ./ /app

#패키지파일들 받기
RUN npm install

#배포버젼으로 설정 - 이 설정으로 환경을 나눌 수 있습니다.
ENV NODE_ENV=production

#서버실행
CMD node main.js

```

# 도커 명령어

```
# 최신 버전으로
docker build -t dockernode .

# 버전 명시
docker build -t dockernode:0.0.2 .
docker build --tag dockernode:0.0.3 .
```

```
도커 빌드후 이미지가 생성되었는지 확인하기
docker images
해당 이미지를 포트설정 및 이름 설정과 함께, 컨테이너를 생성
docker create --name node_server -p 3000:3000 dockernode:0.0.2

컨테이너 확인 ( off )
docker ps -a

docker start node_server
docker ps ( on )
```

# 서버 로직 1.

```js
var express = require("express");
var app = express();
const port = 3000;

var server = app.listen(port, function () {
  console.log("Express server has started on port : " + port);
});

app.get("/", function (req, res) {
  res.send("Hello nodemon with docker");
});
```

# 서버 로직 2.

```js
var cluster = require("cluster");
var os = require("os");
var uuid = require("uuid");
const port = 3000;
//키생성 - 서버 확인용
var instance_id = uuid.v4();

/**
 * 워커 생성
 */
var cpuCount = os.cpus().length; //CPU 수
var workerCount = cpuCount / 2; //2개의 컨테이너에 돌릴 예정 CPU수 / 2

//마스터일 경우
if (cluster.isMaster) {
  console.log("서버 ID : " + instance_id);
  console.log("서버 CPU 수 : " + cpuCount);
  console.log("생성할 워커 수 : " + workerCount);
  console.log(workerCount + "개의 워커가 생성됩니다\n");

  //워커 메시지 리스너
  var workerMsgListener = function (msg) {
    var worker_id = msg.worker_id;

    //마스터 아이디 요청
    if (msg.cmd === "MASTER_ID") {
      cluster.workers[worker_id].send({
        cmd: "MASTER_ID",
        master_id: instance_id,
      });
    }
  };

  //CPU 수 만큼 워커 생성
  for (var i = 0; i < workerCount; i++) {
    console.log("워커 생성 [" + (i + 1) + "/" + workerCount + "]");
    var worker = cluster.fork();

    //워커의 요청메시지 리스너
    worker.on("message", workerMsgListener);
  }

  //워커가 online상태가 되었을때
  cluster.on("online", function (worker) {
    console.log("워커 온라인 - 워커 ID : [" + worker.process.pid + "]");
  });

  //워커가 죽었을 경우 다시 살림
  cluster.on("exit", function (worker) {
    console.log("워커 사망 - 사망한 워커 ID : [" + worker.process.pid + "]");
    console.log("다른 워커를 생성합니다.");

    var worker = cluster.fork();
    //워커의 요청메시지 리스너
    worker.on("message", workerMsgListener);
  });

  //워커일 경우
} else if (cluster.isWorker) {
  var express = require("express");
  var app = express();
  var worker_id = cluster.worker.id;
  var master_id;

  var server = app.listen(port, function () {
    console.log(
      "Express 서버가 " + server.address().port + "번 포트에서 Listen중입니다."
    );
  });

  //마스터에게 master_id 요청
  process.send({ worker_id: worker_id, cmd: "MASTER_ID" });
  process.on("message", function (msg) {
    if (msg.cmd === "MASTER_ID") {
      master_id = msg.master_id;
    }
  });

  app.get("/", function (req, res) {
    res.send(
      "안녕하세요 저는<br>[" +
        master_id +
        "]서버의<br>워커 [" +
        cluster.worker.id +
        "] 입니다."
    );
  });
}
```

# 서버 로직 3

```js
var cluster = require("cluster");
var os = require("os");
var uuid = require("uuid");
const port = 3000;
//키생성 - 서버 확인용
var instance_id = uuid.v4();

/**
 * 워커 생성
 */
var cpuCount = os.cpus().length; //CPU 수
var workerCount = cpuCount / 2; //2개의 컨테이너에 돌릴 예정 CPU수 / 2

//마스터일 경우
if (cluster.isMaster) {
  console.log("서버 ID : " + instance_id);
  console.log("서버 CPU 수 : " + cpuCount);
  console.log("생성할 워커 수 : " + workerCount);
  console.log(workerCount + "개의 워커가 생성됩니다\n");

  //워커 메시지 리스너
  var workerMsgListener = function (msg) {
    var worker_id = msg.worker_id;

    //마스터 아이디 요청
    if (msg.cmd === "MASTER_ID") {
      cluster.workers[worker_id].send({
        cmd: "MASTER_ID",
        master_id: instance_id,
      });
    }
  };

  //CPU 수 만큼 워커 생성
  for (var i = 0; i < workerCount; i++) {
    console.log("워커 생성 [" + (i + 1) + "/" + workerCount + "]");
    var worker = cluster.fork();

    //워커의 요청메시지 리스너
    worker.on("message", workerMsgListener);
  }

  //워커가 online상태가 되었을때
  cluster.on("online", function (worker) {
    console.log("워커 온라인 - 워커 ID : [" + worker.process.pid + "]");
  });

  //워커가 죽었을 경우 다시 살림
  cluster.on("exit", function (worker) {
    console.log("워커 사망 - 사망한 워커 ID : [" + worker.process.pid + "]");
    console.log("다른 워커를 생성합니다.");

    var worker = cluster.fork();
    //워커의 요청메시지 리스너
    worker.on("message", workerMsgListener);
  });

  //워커일 경우
} else if (cluster.isWorker) {
  var express = require("express");
  var app = express();
  var worker_id = cluster.worker.id;
  var master_id;

  var server = app.listen(port, function () {
    console.log(
      "Express 서버가 " + server.address().port + "번 포트에서 Listen중입니다."
    );
  });

  //마스터에게 master_id 요청
  process.send({ worker_id: worker_id, cmd: "MASTER_ID" });
  process.on("message", function (msg) {
    if (msg.cmd === "MASTER_ID") {
      master_id = msg.master_id;
    }
  });

  app.get("/", function (req, res) {
    res.send(
      "안녕하세요 저는<br>[" +
        master_id +
        "]서버의<br>워커 [" +
        cluster.worker.id +
        "] 입니다."
    );
  });
  app.get("/workerKiller", function (req, res) {
    cluster.worker.kill();
    res.send("워커킬러 호출됨");
  });
}
```
