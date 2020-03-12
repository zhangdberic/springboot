# sprng boot test

## 1.安装和配置

### 1.1 pom.xml

```xml
		<!-- spring boot test -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
```

### 1.2 简单的测试用例

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = Z1AutoConfiguration.class)
@ActiveProfiles("test")
public class AESTest {
	
	@Test
	public void test() {
		AES aes = new AES("123456");
		byte[] data = "黑哥".getBytes();
		byte[] encryptedData = aes.encrypt(data);
		byte[] descryptedData = aes.descrypt(encryptedData);
		Assert.assertArrayEquals(data, descryptedData);
	}

}
```

@SpringBootTest源注释，负责@Configuration声明类的加载。

@ActiveProfiles 指定了环境。