#  CopyFileTODB
```java
public static void main(String[] args) throws IOException {

    String copySql = "COPY TABLENAME FROM STDIN DELIMITER E'|' CSV";

    Connection connection = null;
    FileReader fileReader = null;
    long batchStartTime = System.currentTimeMillis();

    try {
      Class.forName("org.postgresql.Driver");
      connection =
          DriverManager.getConnection("jdbc:postgresql://192.168.201.11:5432/postgres?reWriteBatchedInserts=true", "USERNAME", "PASSWORD");
      CopyManager copyManager = new CopyManager((BaseConnection) connection);
      fileReader = new FileReader("csvFilePath");
      Long lineSum = copyManager.copyIn(copySql, fileReader);
      long batchEndTime = System.currentTimeMillis() - batchStartTime;
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      fileReader.close();
    }
    if (connection != null) {
      try {
        connection.close();
      } catch (SQLException e) {
        LOGGER.error("connection close error " + e.getMessage());
      }
    }
  }
```

# CopyStreamTODB
```java
public static void main(String[] args) {

    String copySql =
        "COPY TABLENAME FROM STDIN DELIMITER E'|' CSV";
    Connection connection = null;
    InputStream inputStream = null;
    int batchSize = 10000;
    int totalNumber = 1000000;
    try {
      connection =
          DriverManager.getConnection("jdbc:postgresql://192.168.201.11:5432/postgres?reWriteBatchedInserts=true", "USERNAME", "PASSWORD");
      long startTime = System.currentTimeMillis();
      CopyManager copyManager = new CopyManager((BaseConnection) connection);
      StringBuilder stringBuilder = new StringBuilder();
      for (int i = 1; i <= totalNumber; i++) {
        stringBuilder.append(
            DataUtil.getCurrentTimestamp()
                + "|"
                + DataUtil.getIntValue()
                + "|"
                + DataUtil.getIntValue()
                + "|"
                + DataUtil.getIntValue()
                + "|"
                + DataUtil.getIntValue()
                + "|"
                + DataUtil.getIntValue()
                + "\r\n");
        if (i % batchSize == 0) {
          inputStream =
              new ByteArrayInputStream(stringBuilder.toString().getBytes(StandardCharsets.UTF_8));
          copyManager.copyIn(copySql, inputStream);
          stringBuilder.delete(0, stringBuilder.length());
        }
      }
      // 提交末尾的数据
      inputStream =
          new ByteArrayInputStream(stringBuilder.toString().getBytes(StandardCharsets.UTF_8));
      copyManager.copyIn(copySql, inputStream);
      long spendTime = System.currentTimeMillis() - startTime;
    } catch (SQLException e) {
      e.printStackTrace();
    } catch (IOException e) {
    } finally {
      if (inputStream != null) {
        try {
          inputStream.close();
        } catch (IOException e) {
        }
      }
      if (connection != null) {
        try {
          connection.close();
        } catch (SQLException e) {
        }
      }
    }
  }
```
