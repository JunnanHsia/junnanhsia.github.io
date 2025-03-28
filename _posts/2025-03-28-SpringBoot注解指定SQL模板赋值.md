---
title: SpringBoot注解指定SQL模板给请求参数或响应结果赋值
description: 当主表依赖很多的子表，而项目中又不推荐使用join表来获取子表数据的情况下，或者需要根据请求参数中的某个字段值给其它关联的冗余字段进行赋值，此处给出一种解决方案，可以在接口方法中增加注解，注解中设置特定格式的SQL模板，工具会自动的填充对应的字段值；或者
date: 2025-03-28 16:56:58 +0800
categories: [后端, 高级操作]
tags: [SpringBoot,注解,SQL]
toc: true
comments: true
mermaid: true
math: true
pin: false
image:
  path: /assets/img/posts/reflection.webp
  alt: SpringBoot注解指定SQL模板给请求参数或响应结果赋值
---

## 一、工具类
### 1.注解
```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;


@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FieldSqlReflection {

    /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ 请求参数处理部分配置项 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
    //请求参数处理总开关
    boolean reqEnable() default false;

    //当方法中存在多个参数时,针对哪个字段进行操作的索引,从0开始,默认0
    int reqParamsIndex() default 0;

    /*
     IMPORTANT: 不仔细阅读,报错给你看
         请求参数映射的SQL模板如何定义
         1.需要查询的字段必须用as映射到对应的实体字段名称上(反射取值用),且其中必须要包含目标表的主键字段,主键字段名称固定为id
         2.查询条件中,单个字段时则用=符号后面跟上字段名称,多条件查询时,需要指明in字段的字段名称(本地实体字段名称非数据库字段名称)
         eg.
            1. 单主键单字段查询: 含义=>通过projectId查找对应的orgId值,并设置到对应字段上
             select id,org_id as orgId from zhgd_project where id = #{projectId}
     * */
    String[] reqParamsSqlTemplates() default {};
    /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ 请求参数处理部分配置项 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */


    /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ 响应结果处理部分配置项 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
    //响应结果处理总开关
    boolean respEnable() default false;

    //自定义响应结果的核心字段,开关
    boolean respCustomFieldSpelEnable() default false;

    //自定义响应结果的核心字段,spel表达式, 默认处理: resp为固定的响应对象(名称固定,不可自定义),其中的data.list为自定义的响应结果内部的对象(具体参考spql语法)
    String respCustomFieldSpel() default "#resp.data.list";

    /*
     IMPORTANT: 不仔细阅读,报错给你看
         响应结果处理的SQL模板如何定义
         1.需要查询的字段必须用as映射到对应的实体字段名称上(反射取值用),且其中必须要包含目标表的主键字段,主键字段名称固定为id
         2.查询条件中,单个字段时则用=符号后面跟上字段名称,多条件查询时,需要指明in字段的字段名称(本地实体字段名称非数据库字段名称)
         eg.
            1. 单主键单字段查询: 含义=>通过driverId查找对应的driverName值,并设置到对应字段上
             select driver_id as id,worker_name as driverName from lab_driver_info where driver_id = #{driverId}
            2. 多主键多字段查询: 含义=>通过checkUserId,auditUserId,changeHandleUserId,查找对应的checkUserName,auditUserName,changeHandleUserName的值,并设置到对应字段上
            select id,real_name as checkUserName,real_name as auditUserName,real_name as changeHandleUserName from zhgd_user where id in (#{checkUserId,auditUserId,changeHandleUserId})
            3. 单主键多字段查询: 含义=>通过projectId对应的找到对应projectName,projectTypeName,countryName,provinceName,cityName的值并赋值到对应字段上;
            select zp.id,zp.name as projectName,zd.name as projectTypeName, zdc.name as countryName,zr.name as provinceName,zc.name as cityName from zhgd_project zp left join zhgd_region zr on zr.id = zp.province_id left join zhgd_region zc on zc.id = zp.city_id left join zhgd_dictionary zdc on zp.crountry_code = zdc.code left join zhgd_dictionary zd on zp.en_type = zd.code where zp.id in (#{projectId})
     * */
    String[] respSqlTemplates() default {};

    // 自定义返回结果数据处理的类名
    Class<? extends RespProcessor<?>> respProcessor() default DefaultRespProcessor.class;

    interface RespProcessor<$> {
        //before inflate other field to process data
        default void bfIfProcess($ data) {
        }

        //after inflate other field to process data by default
        void afIfProcess($ data);
    }

    class DefaultRespProcessor implements RespProcessor<Object> {
        public void afIfProcess(Object data) {
            // do nothing
        }
    }
    /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ 响应结果处理部分配置项 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */
}

```

### 2.注解实现类（依赖hutool、lombok、SpingAop）
```java
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.map.MapUtil;
import cn.hutool.core.util.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.annotations.*;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.*;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

/**
 * 在方法中加入该注解,并配置指定格式的SQL模板,自动的填充SQL模板中指定的附属字段的值到返回对象和集合中
 **/
@Slf4j
@Aspect
@RequiredArgsConstructor
@Component
public class FieldSqlReflectionAspect {

    @Mapper
    public interface GeneralMapper {
        @Select("${sql}")
        List<Map<String, Object>> select(@Param("sql") String strSql);

        //当前
        // @Insert("${sql}")
        // int insert(@Param("sql") String strSql);
        //
        // @Delete("${sql}")
        // int delete(@Param("sql") String strSql);
        //
        // @Update("${sql}")
        // int update(@Param("sql") String strSql);
    }


    private final GeneralMapper generalMapper;

    @Pointcut("@annotation(org.tekj.base.aop.FieldSqlReflection)")
    private void cutMethod() {
    }

    /**
     * 环绕通知：灵活自由的在目标方法中切入代码
     */
    @Around("cutMethod()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName();
        FieldSqlReflection fieldReflection = getDeclaredAnnotation(joinPoint);

        /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ 请求参数的处理 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
        if (fieldReflection.reqEnable()) {
            int reqParamsIndex = fieldReflection.reqParamsIndex();
            // 获取方法传入参数
            Object[] params = joinPoint.getArgs();
            if (params != null && params.length > 0) {
                if (reqParamsIndex > 0 && reqParamsIndex < params.length) {
                    // 传入参数中指定索引的参数
                    Object reqParams = params[reqParamsIndex];
                    if (reqParams != null) {
                        String[] reqParamsSqlTemplates = fieldReflection.reqParamsSqlTemplates();
                        if (ArrayUtil.isNotEmpty(reqParamsSqlTemplates)) {
                            for (String reqParamsSqlTemplate : reqParamsSqlTemplates) {

                            }
                        } else {
                            log.warn("ReqFieldReflection: Method:{} 请求参数SQL模板为空,不做处理", methodName);
                        }
                    } else {
                        log.warn("ReqFieldReflection: Method:{} 指定请求参数为null,不做处理");
                    }
                } else {
                    log.warn("ReqFieldReflection: Method:{} 请求参数索引:{} 超出范围,不做处理", methodName, reqParamsIndex);
                }
            } else {
                log.warn("ReqFieldReflection: Method:{} 无请求参数不做处理", methodName);
            }
        }
        /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ 请求参数的处理 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */

        // 执行源方法
        Object proceed = joinPoint.proceed();

        /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ 响应结果的处理 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
        // 获取目标方法的名称,调试日志用
        if (!fieldReflection.respEnable()) {
            log.info("RespFieldReflection: Method:{} 未开启响应结果字段填充功能,不做处理", methodName);
            return proceed;
        }
        if (fieldReflection == null) {
            log.warn("RespFieldReflection: Method:{} 没有RespFieldReflection注解不做处理", methodName);//正常走不到这里
            return proceed;
        }
        if (ObjectUtil.isNull(proceed)) {
            log.warn("RespFieldReflection: Method:{} 目标结果数据为空不做处理", methodName);
            return proceed;
        }

        /**
         整体数据填充逻辑
         0. 在需要处理返回结果的方法上追加@RespFieldReflection注解,根据注解中的使用说明做好配置
         1. 反射获取出需要处理的结果数据,包装成集合对象,主列表数据
         2. 在注解中拿到具体的查询SQL模板集合遍历,根据模板中的指定查询字段和填充字段,从主列表数据中,获取对应的值,填充到SQL模板中,执行SQL模板,获取结果数据;结果数据整理出VLE_MAP,ID_SET_FIELDS_MAP两个值映射对象;
         3. 遍历主列表数据,根据VLE_MAP和ID_SET_FIELDS_MAP,填充目标字段值
         * **/

        //step.1
        List<?> rows = null;
        Object distData = proceed;
        if (fieldReflection.respCustomFieldSpelEnable() && StrUtil.isNotBlank(fieldReflection.respCustomFieldSpel())) {
            //自定义核心字段的映射,使用spel表达式
            try {
                StandardEvaluationContext ctx = new StandardEvaluationContext();
                distData = new Object();
                ctx.setVariable("distData", distData);
                ctx.setVariable("resp", proceed);
                ExpressionParser parser = new SpelExpressionParser();
                Expression expression = parser.parseExpression(fieldReflection.respCustomFieldSpel());
                distData = expression.getValue(ctx);
                if (distData == null) {
                    log.warn("RespFieldReflection: Method:{} 自定义核心响应结果字段映射表达式:{} 解析结果为空,不做处理", methodName, fieldReflection.respCustomFieldSpel());
                    return proceed;
                }
            } catch (Exception e) {
                log.error("RespFieldReflection: Method:{} 自定义核心响应结果字段映射表达式:{} 解析异常,不做处理", methodName, fieldReflection.respCustomFieldSpel(), e);
                return proceed;
            }
        }
        if (distData instanceof Collection) {
            rows = new ArrayList((Collection) distData);
        } else if (distData instanceof Map) {//单个对象Map的情况
            rows = CollUtil.newArrayList((Map) distData);
        } else {//单个对象的情况
            rows = CollUtil.newArrayList(distData);
        }
        if (CollUtil.isEmpty(rows)) {
            log.warn("RespFieldReflection: Method:{} 无目标结果数据集合不做处理", methodName);
            return proceed;
        }

        Object first = CollUtil.getFirst(rows);
        boolean isMap = first instanceof Map;

        //step.2
        //K:需要填充名称的id的字段名称(mainIdField) K:K:主键字段(mainIdField)对应的值(ID) K:V:主键字段(mainIdField)对应的数据对象  K:K:K:主键字段对应的表字段名称 K:K:V:主键字段对应的表字段值
        Map<String, Map<String, Map<String, Object>>> VLE_MAP = new HashMap<>();
        //K:需要填充名称的id的字段名称(mainIdField) V:需要填充值的字段名称的字段集合(不存在不填充值)
        Map<String, List<String>> ID_SET_FIELDS_MAP = new HashMap<>();
        String[] cfSrcQuerySqlTemplates = fieldReflection.respSqlTemplates();
        for (String cfSrcQuerySqlTemplate : cfSrcQuerySqlTemplates) {
            if (StrUtil.isEmpty(cfSrcQuerySqlTemplate) || !cfSrcQuerySqlTemplate.contains("id") || !cfSrcQuerySqlTemplate.contains("#{")) {
                //防御编程,防止SQL模板写错
                continue;
            }
            try {
                //解析SQL模板,处理目标字段和源字段
                String mainIdFieldStr = ReUtil.getGroup1(Pattern.compile("#\\{(.*?)\\}"), cfSrcQuerySqlTemplate);//源字段主键
                List<String> mainIdFields = StrUtil.split(mainIdFieldStr, ',');//存在多个主键id的情况

                List<String> cfDists = ReUtil.findAllGroup1(Pattern.compile("[as,AS] (\\w+)\\s*[\\\\, ]", Pattern.MULTILINE), cfSrcQuerySqlTemplate);//目标字段集合
                cfDists.remove("id");//目标,不应该包含主键
                cfDists.removeAll(mainIdFields);//目标,不应该包含主键

                boolean hasAllField = false;
                if (!isMap) {//校验对象中的字段是否都存在
                    List<String> fieldNames = CollUtil.newArrayList(mainIdFields);
                    fieldNames.addAll(cfDists);
                    hasAllField = fieldNames.stream().allMatch(fieldName -> ReflectUtil.hasField(first.getClass(), fieldName));
                }
                boolean hasFileId = ObjectUtil.isNotEmpty(mainIdFields) && ObjectUtil.isNotEmpty(cfDists)
                        && (isMap || hasAllField);
                if (!hasFileId) {
                    log.warn("RespFieldReflection: Method:{} 自定义字段配置错误: 存在部分自定义的字段定义不存在:{},请检查!", methodName, mainIdFields);
                } else {
                    Set<String> mainIds = new HashSet<>(rows.stream()
                            .filter(ObjectUtil::isNotNull)
                            .flatMap(item -> {
                                if (isMap) {
                                    return mainIdFields.stream()
                                            .map(mainIdField -> MapUtil.getStr((Map<?, ?>) item, mainIdField))
                                            .filter(StrUtil::isNotEmpty);
                                } else {
                                    return mainIdFields.stream()
                                            .map(mainIdField -> ReflectUtil.getFieldValue(item, mainIdField) + "")
                                            .filter(StrUtil::isNotEmpty);
                                }
                            }).collect(Collectors.toSet()));
                    if (CollUtil.isNotEmpty(mainIds)) {
                        String sqlFormat = cfSrcQuerySqlTemplate.replaceAll("#\\{.*?\\}", CollUtil.join(mainIds, ","));
                        try {
                            List<Map<String, Object>> maps = generalMapper.select(sqlFormat);
                            if (CollUtil.isNotEmpty(maps)) {
                                if (!CollUtil.getFirst(maps).containsKey("id")) {
                                    log.warn("RespFieldReflection: Method:{} 查询:{} 自定义字段查询结果中不存在id字段,请检查!", methodName, sqlFormat);
                                } else {
                                    boolean isMultiId = mainIdFields.size() > 1;//多主键id的情况下,默认是主键id和对应的查询字段是一一对应处理的
                                    for (String mainIdField : mainIdFields) {
                                        Map<String, Map<String, Object>> distMap;
                                        if (isMultiId) {
                                            String mainIdName = mainIdField.replace("id", "name").replace("Id", "Name");
                                            distMap = new HashMap<>();
                                            for (Map<String, Object> map : maps) {
                                                if (Objects.isNull(map)) continue;
                                                distMap.put(map.get("id") + "", new HashMap<String, Object>() {{
                                                    put(mainIdName, map.get(mainIdName));
                                                }});
                                            }
                                            VLE_MAP.put(mainIdField, distMap);
                                        } else {
                                            distMap = maps.stream()
                                                    .filter(Objects::nonNull)
                                                    .collect(Collectors.toMap(
                                                            item -> item.get("id") + "",
                                                            item -> item));
                                        }
                                        Map<String, Map<String, Object>> existMap = VLE_MAP.getOrDefault(mainIdField, new HashMap<>());
                                        if (MapUtil.isNotEmpty(existMap)) {
                                            //融合已存在的字段
                                            for (Map.Entry<String, Map<String, Object>> existEntry : existMap.entrySet()) {
                                                if (distMap.containsKey(existEntry.getKey())) {
                                                    Map<String, Object> innerObject = existEntry.getValue();
                                                    innerObject.putAll(distMap.get(existEntry.getKey()));
                                                } else {
                                                    existMap.put(existEntry.getKey(), distMap.get(existEntry.getKey()));
                                                }
                                            }
                                        } else {
                                            existMap.putAll(distMap);
                                        }
                                        VLE_MAP.put(mainIdField, existMap);
                                    }
                                }
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                            log.error("RespFieldReflection: Method:{} 附属字段查询异常:{}", methodName, e.getMessage());
                        }
                    }
                }
                boolean isMultiId = mainIdFields.size() > 1;//多主键id的情况下,默认是主键id和对应的查询字段是一一对应处理的
                for (String mainIdField : mainIdFields) {
                    List<String> distCf = new ArrayList<>();
                    if (isMultiId) {
                        String mainIdName = mainIdField.replace("id", "name").replace("Id", "Name");
                        distCf = cfDists.stream()
                                .filter(item -> ObjectUtil.equal(item, mainIdName))
                                .collect(Collectors.toList());
                    } else {
                        distCf = cfDists;
                    }
                    List<String> orDefault = ID_SET_FIELDS_MAP.getOrDefault(mainIdField, new ArrayList<>());
                    orDefault.addAll(distCf);
                    ID_SET_FIELDS_MAP.put(mainIdField, orDefault);
                }
            } catch (Exception e) {
                e.printStackTrace();
                log.warn("RespFieldReflection: Method:{} 自定义字段存在异常,附属字段查询异常", methodName);
            }
        }

        //自定义数据处理器
        FieldSqlReflection.RespProcessor<Object> rsProcessor = null;
        if (!FieldSqlReflection.DefaultRespProcessor.class.getName().equals(fieldReflection.respProcessor().getName())) {
            rsProcessor = (FieldSqlReflection.RespProcessor<Object>) ReflectUtil.newInstance(fieldReflection.respProcessor());//TODO 针对此类对象做缓存处理,防止每次
        }

        //step.3
        for (Object row : rows) {
            if (rsProcessor != null) {
                rsProcessor.bfIfProcess(row);
            }
            try {
                //自定义字段填充
                if (MapUtil.isNotEmpty(VLE_MAP)) {
                    for (Map.Entry<String, List<String>> cfSrcs : ID_SET_FIELDS_MAP.entrySet()) {
                        String mainIdField = cfSrcs.getKey();
                        if (!VLE_MAP.containsKey(mainIdField)) continue;
                        Map<String, Map<String, Object>> vleMap = VLE_MAP.get(mainIdField);
                        List<String> cfDists = cfSrcs.getValue();
                        for (String cfDist_ : cfDists) {
                            if (isMap) {
                                String cfSrcVle = MapUtil.getStr((Map<?, ?>) row, mainIdField);
                                if (StrUtil.isEmpty(cfSrcVle)) continue;
                                if (vleMap.containsKey(cfSrcVle)) {
                                    Map<String, Object> vle = vleMap.get(cfSrcVle);
                                    if (vle != null) {
                                        ((Map<String, Object>) row).put(cfDist_, vle.get(cfDist_));
                                    }
                                }
                            } else {
                                String cfSrcVle = ReflectUtil.getFieldValue(row, mainIdField) + "";
                                if (StrUtil.isEmpty(cfSrcVle)) continue;
                                if (vleMap.containsKey(cfSrcVle)) {
                                    Map<String, Object> vle = vleMap.get(cfSrcVle);
                                    if (vle != null) {
                                        ReflectUtil.setFieldValue(row, cfDist_, vle.get(cfDist_));
                                    }
                                }
                            }
                        }
                    }
                }
            } catch (Exception e) {
                log.error("AutoCommonBizInfo: Method:{} 自动填充业务数据值异常:{}", methodName, e.getMessage());
                e.printStackTrace();
            }
            if (rsProcessor != null) {
                rsProcessor.afIfProcess(row);
            }
        }
        /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ 响应结果的处理 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */

        return proceed;
    }

    /**
     * 获取方法中声明的注解
     *
     * @param joinPoint
     * @return
     * @throws NoSuchMethodException
     */
    public FieldSqlReflection getDeclaredAnnotation(ProceedingJoinPoint joinPoint) throws NoSuchMethodException {
        // 获取方法名
        String methodName = joinPoint.getSignature().getName();
        // 反射获取目标类
        Class<?> targetClass = joinPoint.getTarget().getClass();
        // 拿到方法对应的参数类型
        Class<?>[] parameterTypes = ((MethodSignature) joinPoint.getSignature()).getParameterTypes();
        // 根据类、方法、参数类型（重载）获取到方法的具体信息
        Method objMethod = targetClass.getMethod(methodName, parameterTypes);
        // 拿到方法定义的注解信息
        FieldSqlReflection annotation = objMethod.getDeclaredAnnotation(FieldSqlReflection.class);
        // 返回
        return annotation;
    }
}

```

## 二、使用
### 1.在需要的方法中增加注解
```java

```
### 2.观察日志和结果
```java

```
