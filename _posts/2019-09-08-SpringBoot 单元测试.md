## SpringBoot 单元测试

用户的行为是无法预料的!!!!


```text
B：Border，边界值测试，包括循环边界、特殊取值、特殊时间点、数据顺序等。
C：Correct，正确的输入，并得到预期的结果。
D：Design，与设计文档相结合，来编写单元测试。
E：Error，强制错误信息输入（如：非法数据、异常流程、非业务允许输入等），并得到预期的结果。
```

### Dao测试

通过 `Idea` 或者 [start.spring.io](https://start.spring.io) 创建一个 `SpringBoot项目` . 添加 `Mybatis` 作为ORM框架.

* `H2` 内存数据库依赖(测试dao)
* `Junit` 单元测试
* `spring-boot-starter-test`

配置 `H2` 数据库信息

```property
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.schema=classpath:db/schema.sql
spring.datasource.data=classpath:db/data.sql
logging.path=${user.home}/open/test
```

在 `src/resources/db` 目录下配置 `schema.sql` 和 `data.sql`. 

> ⚠️ 如果 `data.sql` 中不需要补充数据，这里的配置需要移除，否则会报错. 
> `schema.sql` 需要符合 `H2` 的语法，直接从 `Mysql` 导出的文件不可用.


可以使用 [转换代码](https://gitlab.com/chenshun00/test/blob/master/src/test/java/top/huzhurong/gateway/test/view/TT.java) 将 `Mysql` 文件转换为 `h2`.


测试代码:

```java
    @Autowired
    private UserDao userDao;

    @Test
    public void testGood() {
        userDao.insert();
        userDao.delete();
    }
```

### Service测试

使用 `Juint` 和 `Mockito` 对 `Service` 进行测试，测试的目的是保证每一个service的方法都是正确的。 同时又要满足上述的 `BCDE`.  [Mockito的使用](https://github.com/hehonghui/mockito-doc-zh) 

> 不支持测试 `private` 方法.

#### 被测类中被@Autowired 或 @Resource 注解标注的依赖对象，如何控制其返回值

主要就是如何使用 `Mockito` 对上述注解对象进行 `mock` ， 测试时我们要相信下层代码和其他method提供的返回值都是正确的，每一个测试都只测试一个method。

```java
@SpringBootTest
@RunWith(SpringRunner.class)
@ActiveProfiles("local")
public class UserServiceTest {

    @InjectMocks
    private UserService userService = new UserService();

    @Mock
    private UserDao userDao;
    @Mock
    private TestMockDao testMockDao;

    @Before
    public void setUp() throws Exception {
        // 初始化测试用例类中由Mockito的注解标注的所有模拟对象
        MockitoAnnotations.initMocks(this);
    }
    /**
     * 被测类中被@Autowired 或 @Resource 注解标注的依赖对象，如何控制其返回值
     */
    @Test
    public void result() {
        when(userDao.insert()).thenReturn("陈顺");//mock 返回值
        when(testMockDao.good()).thenReturn("2234"); // mock返回值
        User user = new User();
        user.setUsername("fff");
        String result = userService.result(user);
        assertEquals("陈顺_2234", result);
    }
}
```

#### 被测函数A调用被测类其他函数B，怎么控制函数B的返回值？

此处需要使用到 `Spy` , 请参考 [Mockito Spy 用法](http://youngfor.me/2017/07/30/mockito-spy/) ...
 
```java
 @Test
 public void testOtherReturn2() {
    when(userDao.insert()).thenReturn("陈顺");
    when(testMockDao.good()).thenReturn("2234");
    User user = new User();
    user.setUsername("fff");
    userService = spy(userService);
    //doReturn 开头
    doReturn(Boolean.FALSE).when(userService).otherUsage();
    when(userService.otherUsage()).thenReturn(Boolean.FALSE);
    String result = userService.result(user);
    assertEquals("陈顺_2234", result);
}
```

使用 `spy` 为控制了其他方法到返回值, 每次测试都只测试一个method.

### 使用git hook进行代码检测

```bash
git clone https://gitlab.com/chenshun00/test.git
cd test
cd .git/hooks
mv pre-push.sample pre-push.sample.bak
mv pre-push.sample pre-push
vim pre-push
```

`pre-push` 返回值为 `0` 执行 `git push` 后续动作， 返回值为 `1` ，git push执行中断.
 
```bash
remote="$1"
url="$2"
z40=0000000000000000000000000000000000000000
while read local_ref local_sha remote_ref remote_sha
do
	if [ "$local_sha" = $z40 ]
	then
		:
	else
		if [ "$remote_sha" = $z40 ]
		then
			range="$local_sha"
		else
			range="$remote_sha..$local_sha"
		fi
		commit=`git rev-list -n 1 --grep '^WIP' "$range"`
		if [ -n "$commit" ]
		then
			echo >&2 "Found WIP commit in $local_ref, not pushing"
			exit 1
		fi
	fi
done

#### 这里为补充的代码
echo '开始执行单元测试'
mvn test
if [ $? -ne 0 ];then
        echo '执行mvn test失败，先检查单元测试代码!'
        exit 1
fi

exit 0
```

通过SpringBoot单元测试 + `Gitlab-ci` 就可以完美的执行我们需要的单元测试了. 虽然前期会稍微费一点时间，却会减少线上潜在的小问题. :)

代码地址: [code](https://gitlab.com/chenshun00/test)

#### 参考文章

[阿里巴巴java开发手册#单元测试](https://www.kancloud.cn/kanglin/java_developers_guide/539190)<br>
[Mockito框架中文文档](https://github.com/hehonghui/mockito-doc-zh)<br>
[SpringBoot与JUnit+Mockito 单元测试](https://www.tianmaying.com/tutorial/JunitForSpringBoot)
