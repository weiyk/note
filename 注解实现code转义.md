### 注解实现code转义

#### 1 定义枚举

```java
public enum TradeType {
    /**
     * 充值
     */
    RECHARGE(1, "充值", "Recharge"),
    /**
     * 交易
     */
    TRADE(2, "交易", "Trade");

    private final int code;
    private final String descZh;
    private final String descEn;

    TradeType(int code, String descZh, String descEn) {
        this.code = code;
        this.descZh = descZh;
        this.descEn = descEn;
    }

    public static String getDescByCode(int code, String lang) {
        String result = "";
        for (TradeType tradeType : TradeType.values()) {
            if (code == tradeType.getCode()) {
                if ("zh_CN".equals(lang)) {
                    result = tradeType.getDescZh();
                } else {
                    result = tradeType.getDescEn();
                }
                break;
            }
        }
        return result;
    }
}
```

#### 2 定义转义注解

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Translate {
    /**
     * 转义结果赋值给哪个字段
     */
    String targetField();

    /**
     * 转义的枚举类
     */
    Class<?> enumType();
}
```

#### 3 转义工具类

```java
public class TranslateUtil {
    /**
     * 转义集合
     * @param collection    集合
     * @param lange 语言
     */
    public static <T> void translate(Collection<T> collection, String lange) {
        for (T item : collection) {
            translate(item, lange);
        }
    }

    /**
     * 转义对象
     * @param t 对象
     * @param lang 语言
     */
    public static <T> void translate(T t, String lang) {
        Class<?> clazz = t.getClass();
        Field[] fields = clazz.getDeclaredFields();

        // fieldName ---> Field
        Map<String, Field> fieldMap = new HashMap<>(fields.length);
        for (Field field : fields) {
            fieldMap.put(field.getName(), field);
        }

        try {
            Collection<Field> fieldList = fieldMap.values();
            for (Field field : fieldList) {
                Translate annotation = field.getAnnotation(Translate.class);
                if (annotation == null) {
                    continue;
                }

                Field targetField = fieldMap.get(annotation.targetField());
                if (targetField == null) {
                    continue;
                }

                field.setAccessible(true);
                Object code = field.get(t);
                if (code == null) {
                    continue;
                }

                Class<?> enumClazz = annotation.enumType();
                Method getDescByCodeMethod = enumClazz.getMethod("getDescByCode", int.class, String.class);
                // 字段转义结果
                Object name = getDescByCodeMethod.invoke(enumClazz, code, lang);
                // 赋值给目标字段
                targetField.setAccessible(true);
                targetField.set(t, name);
            }
        } catch (IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
            log.error("转义失败", e);
        }
    }
}
```

#### 4 测试

- 给需要转义的字段增加@Translate注解

```java
@Translate(targetField = "tradeTypeName", enumType = TradeType.class)
private Integer tradeType;
private String tradeTypeName;
```

- 调用转义工具类执行转义

```java
TranslateUtil.translate(pageResult.getRecords(), "zh_CN");
```