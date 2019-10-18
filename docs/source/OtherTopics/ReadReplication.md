# 대체 읽기

Sequelize는 읽기 복제, 즉 SELECT 쿼리를 수행 할 때 연결할 수있는 여러 서버가있는 읽기 복제를 지원합니다. 읽기 복제를 수행 할 때는 하나 이상의 서버를 읽기 전용 복제본으로, 하나의 서버를 쓰기 마스터로 지정하여 모든 쓰기 및 업데이트를 처리하고이를 복제본으로 전파합니다 (실제 복제 프로세스는 `not`라는 점에 유의하십시오) Sequelize에서 처리하지만 데이터베이스 백엔드에서 설정해야합니다).

```js
const sequelize = new Sequelize('database', null, null, {
  dialect: 'mysql',
  port: 3306
  replication: {
    read: [
      { host: '8.8.8.8', username: 'read-username', password: 'some-password' },
      { host: '9.9.9.9', username: 'another-username', password: null }
    ],
    write: { host: '1.1.1.1', username: 'write-username', password: 'any-password' }
  },
  pool: { // If you want to override the options used for the read/write pool you can do so here
    max: 20,
    idle: 30000
  },
})
```

모든 복제본에 적용되는 일반 설정이 있으면 각 인스턴스에 해당 설정을 제공 할 필요가 없습니다. 앞의 코드에서 데이터베이스 이름과 포트는 모든 복제본으로 전파됩니다. 복제본에 대해 사용자 및 비밀번호를 제외하면 동일한 결과가 발생합니다. 각 봅제본에는 다음과 같은 옵션이 있습니다: `host`, `port`, `username`, `passwordd`, `database`.


Sequelize는 풀을 사용하여 복제본에 대한 연결을 관리합니다. 내부적으로 Sequelize는 `pool` 설정을 사용하여 생성 된 두 개의 풀을 유지 관리합니다.

이를 수정하려면 위와 같이 Sequelize를 인스턴스화 할 때 풀을 옵션으로 전달할 수 있습니다.

각 `write` 또는 `useMaster: true` 쿼리는 쓰기 풀을 사용합니다. `SELECT`에는 읽기 풀이 사용됩니다. 읽기 전용 복제본은 기본 라운드 로빈 일정을 사용하여 전환됩니다.
