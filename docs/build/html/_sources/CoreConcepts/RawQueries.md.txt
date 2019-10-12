# 저수준 쿼리

저수준/이미 준비된 SQL 쿼리를 실행하는 것이 더 쉬운 사용사례가 있습니다. `sequelize.query` 함수를 사용하 수 있습니다.

기본적으로 함수는 2개의 인자를 리스트 형태로 반환하며 각 요소는 메타데이터가 포함된 객체입니다. 이것은 원시데이터 이므로 메타데이터는 dialect에따라 다릅니다. 일부 dialect는 배열의 속성으로 결과 개체 "within"에 메타 데이터를 반환합니다. 그러나 구대의 인자는 항상 반환되며, 그러나 MSSQL과 MySQL의 경우 동일한 객체에 대한 두 개를 참조합니다.

```js
sequelize.query("UPDATE users SET y = 42 WHERE x = 12").then(([results, metadata]) => {
  // Results will be an empty array and metadata will contain the number of affected rows.
})
```

메타데이터 접근이 필요 없는경우 쿼리 형식을 전달하여 결과의 형식을 지정하는 방법을 시퀀스에 전달할 수 있습니다. 예를 들어 간단한 조회 쿼리의 경우 다음을 수행 할 수 있습니다.

```js
sequelize.query("SELECT * FROM `users`", { type: sequelize.QueryTypes.SELECT})
  .then(users => {
    // We don't need spread here, since only the results will be returned for select queries
  })
```

몇 가지 다른 쿼리 타입을 사용할 수 있습니다. [더 자세한 코드를 보려면 누르세요.](https://github.com/sequelize/sequelize/blob/master/lib/query-types.js)

두 번째 옵션은 모델입니다. 모델을 전달하면 반환 된 데이터는 해당 모델의 인스턴스입니다.

```js
// Callee is the model definition. This allows you to easily map a query to a predefined model
sequelize
  .query('SELECT * FROM projects', {
    model: Projects,
    mapToModel: true // pass true here if you have any mapped fields
  })
  .then(projects => {
    // Each record will now be an instance of Project
  })
```

쿼리 [API 참조](https://sequelize.org/master/class/lib/sequelize.js~Sequelize.html#instance-method-query)에서 더 많은 옵션을 참조하십시오. 아래 몇 가지 예 :

```js
sequelize.query('SELECT 1', {
  // A function (or false) for logging your queries
  // Will get called for every SQL query that gets sent
  // to the server.
  logging: console.log,

  // If plain is true, then sequelize will only return the first
  // record of the result set. In case of false it will return all records.
  plain: false,

  // Set this to true if you don't have a model definition for your query.
  raw: false,

  // The type of query you are executing. The query type affects how results are formatted before they are passed back.
  type: Sequelize.QueryTypes.SELECT
})

// Note the second argument being null!
// Even if we declared a callee here, the raw: true would
// supersede and return a raw object.
sequelize
  .query('SELECT * FROM projects', { raw: true })
  .then(projects => {
    console.log(projects)
  })
```

## "Dotted" 속성


테이블의 속성 이름에 점이 포함되어 있으면 결과 개체가 중첩됩니다. 이것은 후드에서 dottie.js를 사용하기 때문입니다.

```js
sequelize.query('select 1 as `foo.bar.baz`').then(rows => {
  console.log(JSON.stringify(rows))
})
```

```js
[{
  "foo": {
    "bar": {
      "baz": 1
    }
  }
}]
```

## 대체자

쿼리에서 대체는 이미 정의된 매개변수(`:`로 시작)를 사용하거나 이미 정의되지 않은 `?`로 표시하는 방법이 있습니다. 교체는 옵션 개체에 전달됩니다.

* 배열이 전달되면 `?` 배열에 나타나는 순서대로 교체됩니다
* 객체가 전달되면 `: key`가 해당 객체의 키로 대체됩니다. 개체에 쿼리에서 찾을 수없는 키가 포함 된 경우 또는 그 반대의 경우 예외가 발생합니다.

```js
sequelize.query('SELECT * FROM projects WHERE status = ?',
  { replacements: ['active'], type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})

sequelize.query('SELECT * FROM projects WHERE status = :status ',
  { replacements: { status: 'active' }, type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})
```

배열 교체는 자동으로 처리되며 다음 쿼리는 상태가 값 배열과 일치하는 프로젝트를 검색합니다.

```js
sequelize.query('SELECT * FROM projects WHERE status IN(:status) ',
  { replacements: { status: ['active', 'inactive'] }, type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})
```


와일드 카드 연산자 `%`를 사용하려면 이를 대체 문자에 추가하십시오. 다음 쿼리는 이름이 'ben'으로 시작하는 사용자를 찾습니다.

```js
sequelize.query('SELECT * FROM users WHERE name LIKE :search_name ',
  { replacements: { search_name: 'ben%'  }, type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})
```

## 파라미터 바인드

파라미터 바인드는 대체자와 비슷합니다. 대체자는 SQL 쿼리 텍스트 외부의 데이터베이스로 전송되는 반면, 바인드 매개 변수는 데이터베이스로 전송되기 전에 대체가 이스케이프되고 쿼리에 삽입되어 제외됩니다. 쿼리는 바인드 매개 변수 또는 대체자를 가질 수 있습니다. 바인드 매개변수는 $1, $2, ...(숫자) Ehsms $key (영숫자)로 참조합니다. 이것은 다른 데이터에 의존적이지 않습니다.

* 배열이 전달되면 $1은 배열의 첫 번째 요소에 바인딩됩니다 (bind [0]).
* 객체가 전달되면 $key는 object ['key']에 바인딩됩니다. 각 키는 숫자가 아닌 문자로 시작해야합니다. object ['1']이 있어도 $1은 유효한 키가 아닙니다.
* 두 경우 모두 $$를 사용하여 리터럴 $ 기호를 이스케이프 할 수 있습니다.

배열이나 객체는 모든 바인딩 값을 포함해야합니다. 그렇지 않으면 Sequelize에서 예외가 발생합니다. 이는 데이터베이스가 바인드 된 매개 변수를 무시할 수있는 경우에도 적용됩니다.

데이터베이스에 이에 대한 추가 제한 사항이 추가 될 수 있습니다. 바인드 매개 변수는 SQL 키워드 나 테이블 또는 열 이름이 될 수 없습니다. 인용 된 텍스트 나 데이터에서도 무시됩니다. PostgreSQL에서는 `$1::varchar` 컨텍스트에서 형식을 유추 할 수없는 경우 형식을 변환해야 할 수도 있습니다.

```js
sequelize.query('SELECT *, "text with literal $$1 and literal $$status" as t FROM projects WHERE status = $1',
  { bind: ['active'], type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})

sequelize.query('SELECT *, "text with literal $$1 and literal $$status" as t FROM projects WHERE status = $status',
  { bind: { status: 'active' }, type: sequelize.QueryTypes.SELECT }
).then(projects => {
  console.log(projects)
})
```