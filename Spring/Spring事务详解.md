### 问题

同事在项目中配置了三个数据源和事务管理器，但是数据源是属于同一个MySQL实例，只是不同的数据库，如下配置

```java
/**
	* spring.datasource.hikari.db01.jdbcUrl: 127.0.0.1:3306/test
	*/
@Bean("db01")
@ConfigurationProperties(prefix = "spring.datasource.hikari.db01")
public DataSource dbOne() {
	return DataSourceBuilder.create().type(HikariDataSource.class).build();
}

@Bean("sqlSessionFactory01")
public SqlSessionFactory sqlSessionFactory01(@Qualifier("db01") DataSource dataSource) throws Exception {
        final SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        return bean.getObject();
}

@Bean("transactionManager01")
public PlatformTransactionManager transactionManager01(@Qualifier("db01") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
}

/** split line **/

/**
	* spring.datasource.hikari.db02.jdbcUrl: 127.0.0.1:3306/app
	*/
@Bean("db02")
@ConfigurationProperties(prefix = "spring.datasource.hikari.db02")
public DataSource dbTwo() {
    return DataSourceBuilder.create().type(HikariDataSource.class).build();
}

@Bean("sqlSessionFactory02")
public SqlSessionFactory sqlSessionFactory02(@Qualifier("db02") DataSource dataSource) throws Exception {
    final SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
    bean.setDataSource(dataSource);
    return bean.getObject();
}

@Bean("transactionManager02")
public PlatformTransactionManager transactionManager02(@Qualifier("db02") DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

他在使用过程中，发现出现了类似分布式事务的现象，即嵌套的事务没有正常回滚，所有我做了相关测试，得到如下结论

> ```latex
> 经过测试，同个MySQL实例，多个数据库之间是同一个事务，无需指定多个数据源和事务管理器。
> 只需要指定一个数据源、事务管理器即可，只不过在写mapper的SQL时，表名前加上库名，例如app.table test.table。
> 而且即使有多个数据源，也需要写上表名，因为默认使用的是配置文件中按顺序排第一的那个数据连接。
> ```

### 第二个问题

得出以上结论后，顺便做了一下嵌套事务的测试，发现事务传播行为好像不太符合预期，在一个方法里调用同一个Service里的另外一个方法，外部方法不加事务，内部方法添加@Transactional，异常外部不会回滚，于是又做了一系列测试，并查阅文档，得出以下结论

**示例代码**

```java
@Service
@Slf4j
public class TestTransactionService {

    @Resource
    ScoreMapper scoreMapper;

    @Resource
    StudentMapper studentMapper;

    @Transactional(rollbackFor = Exception.class)
    public int addScore(Score score, Student student) {
        TransactionDebugUtil.transactionRequired("addScore");
        int i = scoreMapper.addScore(score);
        log.info("sco i= "+i);
        i = i + addStudent(student);

        int j = 1/ 0;
        return i;
    }

    @Transactional(rollbackFor = Exception.class, propagation =
            Propagation.REQUIRES_NEW)
    public int addStudent(Student student) {
        TransactionDebugUtil.transactionRequired("addStudent");

        final int i = studentMapper.addStudent(student);
        log.info("stu i= "+i);

        return i;
    }
}
```



**结论**

>**1、addStudent()会不会创建一个新事务？** 
>
>不会创建。仔细查看了日志，没有找到类似creating new transaction的输出，应该是因为在同一个Service类中，Spring并不重新创建新事务，如果是两不同的Service，就会创建新事务了。 
>
>**那么为什么spring只对跨Service的方法才生效？ **
>Debug代码发现跨Service调用方法时，都会经过org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor.intercept()方法，只有经过此处，才能对事务进行控制。 
>
>**2、不同的Service调用方法时：**
>
>如果被调用方法是Propagation.REQUIRES_NEW，被catch后不抛出，事务可以正常提交； 
>如果被调用方法是Propagation.REQUIRED，被catch后不抛出，后面的代码虽然可以执行下去，但最终还是会分出rollback-only异常；
>
>**3、同一个Service中调用方法时：**
>
>不论注解是Propagation.REQUIRES_NEW 还是 Propagation.REQUIRED，
>其结果都是一样的，就是都被忽略掉了，等于没写。
>当其抛出异常时，只需catch住不抛出，事务就可以正常提交。
>
>**4、同一个Service中调用方法时：**
>
>外部方法没有使用@Transactional，内部方法同样没有事务。外部使用了@Transactional，内部方法无论有没有注解，都会有事务。
>
>**5、不支持非public**
>Spring AOP不支持非public方法增强，与自调用类似，也是动态代理无法解决的盲区。
>
>从代码分析原因是TransactionInterceptor拦截器会间接调用到一个AbstractFallbackTransactionAttributeSource里面的computeTransactionAttribute方法，通过这个方法来获取注解上的各个属性，同时判断该方法是否为public，若不是public则不会读取注解属性，返回null。

**原因**

Spring事务采用AOP的方式，调用同类的方法时，是否有事务由第一个方法决定，因为后续调用的是同一个代理对象的其他方法，不会再生成代理对象了。也就是，如果第一个方法有事务，则其他方法都有事务，且是同一个事务。如果第一个方法没有事务，则后续所有方法都没有事务。

**解决方法**

由于是通过AOP代理实现，所有只需要将被调用的其他方法写在不同的@Service里就可以了，这样会生成新的代理对象，事务就可以按照传播行为进行传递。