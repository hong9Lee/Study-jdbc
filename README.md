# Study-jdbc
학부생 시절 사용했던 순수 JDBC 기술에 대한 기억이 가물가물해 인터넷 강의를 통해 재학습



## 커넥션 연결
- 커넥션을 연결하기 위해 매번 복잡한 과정을 거친다
```
public static Connection getConnection() {
  try {
    Connection connection = DriverManager.getConnection(ConnectionConst.URL, ConnectionConst.USERNAME, ConnectionConst.PASSWORD);
    log.info("get connection={}, class={}", connection, connection.getClass());
    return connection;
  } catch (SQLException e) {
    throw new IllegalStateException();
  }
}

-- service 
String sql = "select ...";

Connection con = null;
PreparedStatement pstmt = null;
ResultSet rs = null;

try {
  con = getConnection();
  pstmt = con.prepareStatement(sql);
  pstmt.setString(1, memberId);
  ...

  rs = pstmt.executeQuery();
  if (rs.next()) {
    ...
  } catch (SQLException e) {
    ...
  } finally {
    close(con, pstmt, rs);
}
```

## 커넥션풀 사용  
- 커넥션을 획득하기 위한 복잡한 과정을 해결하기 위해 커넥션을 미리 생성해두고 사용하는 커넥션 풀을 사용
```
void dataSourceConnectionPool() throws SQLException, InterruptedException {

  HikariDataSource dataSource = new HikariDataSource(); 
  dataSource.setJdbcUrl(URL);
  dataSource.setUsername(USERNAME); 
  dataSource.setPassword(PASSWORD); 
  dataSource.setMaximumPoolSize(10); 
  dataSource.setPoolName("MyPool");
  useDataSource(dataSource);

}

private void useDataSource(DataSource dataSource) throws SQLException {
  Connection con1 = dataSource.getConnection();
  log.info("connection={}, class={}", con1, con1.getClass());
}

```
## 트랜잭션
- 데이터베이스에서 완전하게 처리되어야 하는 작업의 단위  
계좌이체 거래 시 주고받는쪽이 로직이 모두 성공한다면 커밋, 그렇지 않다면 롤백 되어야 한다.
```
-- 1. 수동 커밋으로 트랜젝션 테스트
try {
    con.setAutoCommit(false); // 트랜잭션 시작
    bizLogic(con, fromId, toId, money); //비지니스 로직 시작
    con.commit(); // 성공 시 커밋
} catch (Exception e) {
    con.rollback(); // 실패 시 롤백
    ...
  } finally {
    release(con);
}

private void release(Connection con) {
    con.setAutoCommit(true); // 커넥션 풀 반환 고려
    ...     
}

-- 2. Spring에서 제공하는 트랜잭션 인터페이스 사용

private final PlatformTransactionManager transactionManager;

// 트랜잭션 시작
TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
try {
  //비지니스 로직 시작
  bizLogic( fromId, toId, money);
  transactionManager.commit(status); // 성공 시 커밋
} catch (Exception e) {
  transactionManager.rollback(status); // 실패 시 롤백
  ...
}

-- 3. 트랜잭션 시작, 종료와 같은 반복되는 소스를 제거하기위해 트랜잭션 템플릿 사용
...
this.txTemplate = new TransactionTemplate(transactionManager);
txTemplate.executeWithoutResult((status) -> {
    try {
      //비즈니스 로직
      bizLogic(fromId, toId, money);
    } catch (SQLException e) {
      ...
}
});

-- 4. 서비스 계층에 순수한 비지니스 로직만 남기기 위해 @transaction 어노테이션 사용
@Transactional
public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    bizLogic(fromId, toId, money);
}

```
