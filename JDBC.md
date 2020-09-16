# JDBC

## JDBC工作流程

![image-20200624183202143](/Users/bill/Library/Application Support/typora-user-images/image-20200624183202143.png)

##### 尝试获取一个Connection对象

```java
// 方式一
@Test
public void getSQLConnection1() throws SQLException {
    Driver driver = new com.mysql.jdbc.Driver();
    String url = "jdbc:mysql://localhost:3306/users01";
    Properties info = new Properties();
    info.setProperty("user", "root");
    info.setProperty("password","123456");
    Connection conn = driver.connect(url, info);
    System.out.println(conn);
}

// 方式二：对方式一的迭代
    @Test
    public void getSQLConnection2() throws SQLException, ClassNotFoundException, IllegalAccessException, InstantiationException {
        // 1. 获取Driver实现类的对象，利用反射
        Class klass = Class.forName("com.mysql.jdbc.Driver");
        Driver driver = (Driver) klass.newInstance();

        // 2. 提供要连接的数据信息
        String url = "jdbc:mysql://localhost:3306/users01";
        Properties info = new Properties();
        info.setProperty("user", "root");
        info.setProperty("password", "123456");

        // 3. 获取连接
        Connection conn = driver.connect(url, info);
        System.out.println(conn);
    }

    // 方式三：继续迭代，使用DriverManager来替换Driver
    @Test
    public void getSQLConnection3() throws SQLException, ClassNotFoundException, IllegalAccessException, InstantiationException {

        // 1. 提供数据库信息
        String url = "jdbc:mysql://localhost:3306/users01";
        Properties info = new Properties();
        info.setProperty("user", "root");
        info.setProperty("password", "123456");

        // 2.1 获取Driver实现类的对象，利用反射
        Class klass = Class.forName("com.mysql.jdbc.Driver");
        Driver driver = (Driver) klass.newInstance();
        // 2.2 注册驱动
        DriverManager.registerDriver(driver);

        // 3. 获取连接
        Connection conn = DriverManager.getConnection(url, info);
        System.out.println(conn);
    }

    // 方式四：优化方式三
    @Test
    public void getSQLConnection4() throws SQLException, ClassNotFoundException, IllegalAccessException, InstantiationException {

        // 1. 提供数据库信息
        String url = "jdbc:mysql://localhost:3306/users01";
        Properties info = new Properties();
        info.setProperty("user", "root");
        info.setProperty("password", "123456");

        // 2.1 获取Driver实现类的对象，利用反射
        Class.forName("com.mysql.jdbc.Driver");
        //发现，MySQL的Driver实现类中，存在静态代码块，
        // 在这个类加载的时候就已经注册完成了

        // 3. 获取连接
        Connection conn = DriverManager.getConnection(url, info);
        System.out.println(conn);
    }
```

对于方法四的思考，发现MySQL的Driver实现类中存在静态代码块，完成注册。

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

实际上，对于MySQL的Driver来说，加载类     `Class.forName("com.mysql.jdbc.Driver")`也可以省略，这是因为在加载其驱动的时候就加载了这个类，自然就注册了。

###### **获取连接的最终版**

```java
    public void getConnection() throws IOException, ClassNotFoundException, SQLException {
        InputStream inputStream = ConnectionTest.class.getClassLoader().getResourceAsStream("jdbc.properties");

        Properties props = new Properties();
        props.load(inputStream);

        String user = props.getProperty("user");
        String password = props.getProperty("password");
        String url = props.getProperty("url");
        String driverClass = props.getProperty("driverClass");

        Class.forName(driverClass);

        Connection conn = DriverManager.getConnection(url, user, password);
        System.out.println(conn);
    }
```

其中配置文件jdbc.properties为：

```properties
user=root
password=123456
url=jdbc:mysql://localhost:3306/users01
driverClass=com.mysql.jdbc.Driver
```

