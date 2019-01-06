# AOP+动态代理+ThreadLocal改造权限管理

### 1.背景
  公司用着一套比较垃圾的敏捷开发框架，由于很多开发没有按照原有的框架模式开发功能(其实是这个框架比较垃圾，无法满足业务)。原有的项目是全用Hibernate来开发的，项目引入了jdbcTemplate来操作一个sql，原有的权限控制无法注入到jdbcTemplate里，需要改造。

### 2.调研
通过看原项目的源代码，发现原项目是通过Hibernate的过滤器来完成了一些sql注入，按照原项目的思路，通过新建表的一些固定系统变量字段及权限相关的几张表通过sql拼接完成的比较灵活的权限控制。那么我们在兼容原有框架的基础上，只需要按照同样的方式通过jdbcTemplate的sql拼接就可以实现同样的功能。

### 3.思路
在不修改原有jdbcTemplate的DAO抽象类的基础上完成功能拓展，首先想到的就是代理。又由于原有的权限是跟菜单及url绑定到一起的，我们要兼容的话就得采用同样的方式，DAO与Controller不再同一层，要完成url参数的传递，可以通过request或者session来完成，request的话需要一直向下传递未免代码侵入性太强，不可取；session的话，需要维护每个用户的jsessionID太麻烦了；那么我们还有什么可以用的呢？没错，ThreadLocal，在Spring中，如果不做异步的配置的话，一个请求过来，实际上是由一个线程去完成并返回的。用ThreadLocal的话也不用去维护Session，并且根本不存在多线程并发的问题。url参数传递有思路了，项目中有的是用原生框架写的，权限配置可以直接使用，有的没有，这方面的话，采用自定义注解+AOP来实现定制化权限管理，只对我们需要的(即jdbcTemplate方式)去拼接。

### 4.撸代码
思路确定了，开冲！:smile: :smile: :smile:
首先构建基础组件：

------------


  *  ①Thread相关代码*
  ```Java
  /**
 * 本地缓存工具类
 * @version 1.0.0
 * @Description TODO
 * @createTime 2019年01月05日
 */
public class TSDataRuleLocal {
   //这边是静态的保证在一次请求中使用的是同一个ThreadLocal
    private static ThreadLocal<Map<String, Object>> local = new ThreadLocal<>();

    public static void setTSDataRules(Map<String, Object> map) {
        local.set(map);
    }

    public static Map<String, Object> getTSDataRules() {
        return local.get();
    }

    public static void clear() {
        local.remove();
    }
}

  ```
  上面的本地缓存类就是简单的构建了一个map结构的ThreadLocal，主要是为了传递请求中相关的参数使用，有关ThreadLocal的介绍google一搜一大把，这边就不做赘述。

------------



  *  ②自定义注解相关代码*
  ```Java
  /** jdbc方式数据权限标记注解
 * @version 1.0.0
 * @Description TODO
 * @createTime 2019年01月04日
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataPermission {
    /**
     * 需要拦截的url，填写 localhost:8080之后的全部url不包含参数
     * @return
     */
    String value();

    /**
     * 查询语句的表名，有别名的话传别名
     * @return
     */
    String tableName();
}
  ```
自定义注解主要是为了标记需要处理权限的Controller，里面的两个成员变量注释都有，url是为了去数据库查询该请求配置的数据权限规则；tableName是为了拼接sql的时候指定相关的表，因为并不是所有的查询语句都是单表的，主要用于多表查询的时候指定该拼接那个主表的权限字段。这边可以根据需求灵活构建。
------------

  *  ③自定义注解相关代码*
```Java
/**
 * 数据权限缓存切面类
 * @version 1.0.0
 * @Description TODO
 * @createTime 2019年01月04日
 */
@Component
@Aspect
public class AspectDataPermission {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private SystemService systemService;

    @Pointcut("@annotation(com.ztocwst.jingtian.common.util.DataPermission)")
    public void section() {
        logger.info("切面表达式info....");
    }

    @Before("section()")
    public void befor(JoinPoint jp) {
        try {
            Map<String, Object> map = new HashMap<>();
            //存储有效的数据规则
            List<TSDataRule> tmp = new ArrayList<>();
            Object target = jp.getTarget();
            String name = jp.getSignature().getName();
            Method[] methods = target.getClass().getMethods();
            String value = null;
            String tableName = null;
            for (Method method : methods) {
                if (method.getName().equals(name)) {
                    DataPermission annotation = method.getAnnotation(DataPermission.class);
                    value = annotation.value();
                    tableName = annotation.tableName();
                    List<TSFunction> list = systemService.findByQueryString("from TSFunction WHERE functionType=1 AND functionUrl='" + value + "'");
                    if (CollectionUtils.isEmpty(list)) {
                        throw new RuntimeException("Url:" + value + "   权限菜单查询结果为空");
                    }
                    //获取配置好的权限菜单
                    TSFunction function = list.get(0);
                    String id = function.getId();
                    //获取该权限菜单对应的数据权限规则
                    List<TSDataRule> tSDataRuleList = systemService.findByQueryString("from TSDataRule WHERE TSFunction.id ='" + id + "'");
                    //获取有效的数据规则
                    tSDataRuleList.forEach(t -> {
                        List<TSRoleFunction> TSRoleFunctionList = systemService.findByQueryString("from TSRoleFunction WHERE TSFunction.id ='" + id + "'" + "AND dataRule LIKE ('%" + t.getId() + "%')");
                        if (CollectionUtils.isNotEmpty(TSRoleFunctionList)) {
                            tmp.add(t);
                        }
                    });
                }
            }
            check(tableName,value,tmp);
            //存储所需数据
            map.put("tableName",tableName);
            map.put("data",tmp);
            TSDataRuleLocal.setTSDataRules(map);
        } catch (Exception e) {
            logger.error("数据权限切面异常", e);
        }
    }

    private void check(String tableName, String value, List<TSDataRule> tmp){
        if(StringUtils.isEmpty(tableName)){
            throw new RuntimeException("Url:" + value + "   的权限主表属性为空");
        }
        if(CollectionUtils.isEmpty(tmp)){
            throw new RuntimeException("Url:" + value + "   有效的权限数据规则为空");
        }
    }
```

  切面主要是为了拿到带有自定义注解的方法上面的一些值(url和tableName)其他代码就是一些查询菜单数据权限的业务代码以及一些检查，可以直接忽略。主要是TSDataRuleLocal.setTSDataRules(map); 这行代码将要拼接的sql的信息存入ThreadLocal在我们接下来的代理类中要使用。
------------

  *  ④动态代理相关代码*

```Java
/**
 * GenericBaseCommonDao jdbc方法的代理类
 * @version 1.0.0
 * @Description TODO
 * @createTime 2019年01月04日
 */
@Repository
public class ProxyGBCDHandler implements InvocationHandler {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private IGenericBaseCommonDao genericBaseCommonDao;
    private static final String SYS_USER = "#{sys_user_code}";
    private static final String SYS_ORG = "#{sys_org_code}";

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        try {
            String name = method.getName();
            //没有加权限配置不处理
            if (null != TSDataRuleLocal.getTSDataRules()) {
                //只处理jdbc的方法 目前每一个代理的方法其实都是jdbc方法
                if (name.toLowerCase().contains("jdbc") && null != args && args.length != 0) {
                    //处理sql  目前所有方法第一个参数都是String类型的sql 以后新加的方法也要按照命名规则和第一个参数为sql的方式去写
                    String sql = (String) args[0];
                    //获取当前的数据规则及权限主表
                    Map<String, Object> map = TSDataRuleLocal.getTSDataRules();
                    String tableName = (String) map.get("tableName");
                    List<TSDataRule> rules = (List<TSDataRule>) map.get("data");
                    //获得权限规则sql条件
                    String ruleSql = ruleToSqlString(rules);
                    String newSql = buildSql(sql, ruleSql, tableName);
                    //替换原来的参数列表
                    args[0] = newSql;
                }
            }
        } catch (Exception e) {
            logger.error("代理jdbc执行异常",e);
        }
        Object invoke = method.invoke(genericBaseCommonDao, args);
        return invoke;
    }

    /**
     * 拼装权限相关sql
     *
     * @param sql 原始sql
     * @param rule 拼接的权限sql条件
     * @param tableName 权限相关的主表
     * @return
     */
    private String buildSql(String sql, String rule,String tableName) {
        String[] split = sql.split("(?i)where");
        StringBuffer sb = new StringBuffer();
        if (split.length == 1) {
            sb.append(sql);
            sb.append(" WHERE "+tableName+"."+rule);
        } else {
            sb.append(split[0]);
            String a = split[1];
            //有1=1的情况
            if (a.trim().contains("1 = 1")) {
                String[] split2 = a.split("1 = 1");
                sb.append(" WHERE 1=1 AND "+tableName+"."+rule);
                sb.append(split2[1]);
            }
            if (a.trim().contains("1=1")) {
                String[] split2 = a.split("1=1");
                sb.append(" WHERE 1=1 AND "+tableName+"."+rule);
                sb.append(split2[1]);
            }
            if (!a.trim().contains("1=1") && !a.trim().contains("1 = 1") && a.trim().contains("and")) {
                sb.append("WHERE "+tableName+"."+rule+" AND");
                sb.append(a);
                //没有 1=1 有and
                String[] ands = a.split("and");
                System.out.println(Arrays.toString(ands));

            }
        }
        return sb.toString();
    }


    /**
     * 规则转换为sql字符串
     *
     * @param rules
     * @return
     */
    private String ruleToSqlString(List<TSDataRule> rules) {
        StringBuffer sb = new StringBuffer();
        rules.forEach(t -> {
            //得到数据库字段名称
            String column = StringUtil.humpToUnderline(t.getRuleColumn());
            sb.append(column.toLowerCase());

            if (t.getRuleConditions().toUpperCase().equals("LIKE")) {
                //拼接权限模糊查询条件
                sb.append(" " + t.getRuleConditions() + " ('");
                if(SYS_USER.equals(t.getRuleValue().trim())){
                    //获取用户登陆账号  用于只能看见自己录入的数据
                    String userCode = ResourceUtil.getUserSystemData(DataBaseConstant.SYS_USER_CODE);
                    sb.append(userCode + "') ");
                }
                if(SYS_ORG.equals(t.getRuleValue().trim())){
                    //获取用户登陆账号  用于只能看见自己录入的数据
                    String userCode = ResourceUtil.getUserSystemData(DataBaseConstant.SYS_ORG_CODE);
                    sb.append("%"+userCode + "%') ");
                }

            } else {
                //拼接权限精确查询条件
                sb.append(" " + t.getRuleConditions() + " '");
                if(SYS_USER.equals(t.getRuleValue().trim())){
                    //获取用户登陆账号  用于只能看见自己录入的数据
                    String userCode = ResourceUtil.getUserSystemData(DataBaseConstant.SYS_USER_CODE);
                    sb.append(userCode + "' ");
                }
                if(SYS_ORG.equals(t.getRuleValue().trim())){
                    //获取用户登陆账号  用于只能看见自己录入的数据
                    String userCode = ResourceUtil.getUserSystemData(DataBaseConstant.SYS_ORG_CODE);
                    sb.append(userCode + "' ");
                }

            }

        });
        return sb.toString();
    }

}
```
上面的动态代理是在Spring中的实现方式，网上很多动态代理都是在纯Java环境中的动态代理。这边主要是将代理类声明为SpringBean对象交给Spring去管理，在需要的地方直接注入使用就ok。其他逻辑都是拼接sql的业务逻辑，可以直接忽略。genericBaseCommonDao这个bean就是前面说的JDBC DAO的抽象类，也就是我们这边要代理的具体对象。这边将sql处理之后再交给jdbcTemplate去执行sql。

------------

  *  ⑤原DAO抽象类的修改*

```Java

	//代理执行jdbc相关操作
	@Autowired
	private ProxyGBCDHandler proxyGBCDHandler;
	
	@Transactional(readOnly = true)
	public List<Map<String, Object>> findForJdbc(String sql, Object... objs) {
		IGenericBaseCommonDao service = (IGenericBaseCommonDao) Proxy.newProxyInstance(commonDao.getClass().getClassLoader(), new Class[]{IGenericBaseCommonDao.class}, proxyGBCDHandler);
		return service.findForJdbc(sql, objs);
	}

```

这边有大概七八个方法都是同样的操作(感觉这边还可以重构下)，代码如上首先将我们的代理类注入，然后通过 Proxy.newProxyInstance方法来执行动态代理。这边感觉写的并不是很好，后续优化了的话再贴优化后的代码。

------------

  *  ⑥最后别忘了在请求返回之前将ThreadLocal清空*

```Java

public class AuthInterceptor implements HandlerInterceptor {

	/**
	 * 在controller后拦截
	 */
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object object, Exception exception) throws Exception {
		if(null != TSDataRuleLocal.getTSDataRules()){
			TSDataRuleLocal.clear();
		}
	}
}
```


------------

以上，目前使用没发现什么问题，不喜勿喷，有不同的想法欢迎发issuse
