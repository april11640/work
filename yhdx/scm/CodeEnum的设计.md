1、原因

通常枚举在传输或存储时，可以用它的ordinal表示；但是ordinal不太可靠，因为它记录的是枚举类型声明的位置；假如不小心调整了枚举类型声明的位置，则ordinal的值也发生了变化，那么历史记录的数据就是错误的。因此需要设计一个枚举的根类，它能明确枚举类型传输或存储过程中的值，并且不会因为调整枚举类型声明的位置而引起历史记录的错误。

2、CodeEnum接口

```java
public interface CodeEnum<E extends Enum<E>> {    
    /**     
     * 枚举类型的整形值  
     * @return     
     */   
    int getCode();
}
```

java的枚举实际上就是类，所以系统创建的类都实现此接口，可以定义系统枚举统一的类型值表示方式。

3、枚举实现示例

```java
public enum SexEnum implements CodeEnum {
    
	MALE(0),
    FEMALE(1),
    UNKNOWN(2);
    
    private int code;

    SexEnum(int code) {
        this.code = code;
    }

    @Override
    public int getCode() {
        return code;
    }
    
}
```

4、Spring REST中的枚举JSON转换处理

序列化：

```java
@JsonComponent
public class CodeEnumJsonSerializer<E extends Enum<E>> extends JsonSerializer<CodeEnum<E>> {  
    
    @Override    
    public void serialize(CodeEnum<E> value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {  
        gen.writeNumber(value.getCode());    
    }
    
}
```

反序列化：

```java
@JsonComponent
public class CodeEnumJsonDeserializer<E extends Enum<E>> extends JsonDeserializer<E> implements ContextualDeserializer {  
    
    private Class<E> clazz;    
    
    public CodeEnumJsonDeserializer() {    
    }    
    
    public CodeEnumJsonDeserializer(Class<E> clazz) {        
        this.clazz = clazz;    
    }    
    
    @Override    
    public E deserialize(JsonParser p, DeserializationContext ctxt) throws IOException,            JsonProcessingException {        
        JsonToken currJsonToken = p.getCurrentToken();       
        if (JsonToken.VALUE_NULL.equals(currJsonToken)) {       
            return null;        
        }       
        int code = p.getIntValue();        
        E[] enumConstants = clazz.getEnumConstants();      
        for(E item : enumConstants) {            
            if(Objects.equals(code, ((CodeEnum)item).getCode())) {      
                return item;           
            }       
        }       
        return null;    
    } 
    
    @Override    
    public JsonDeserializer<?> createContextual(DeserializationContext deserializationContext, BeanProperty beanProperty) throws JsonMappingException {     
        Class<E> clazz = (Class<E>) deserializationContext.getContextualType().getRawClass();       
        return new CodeEnumJsonDeserializer<>(clazz);  
    }
    
}
```

5、Mybatis的枚举转换处理

```java
public class CodeEnumTypeHandler<E extends Enum<E> & CodeEnum<E>> extends BaseTypeHandler<CodeEnum> {   

    private Class<CodeEnum> type;   
    private Map<Integer, CodeEnum> cache;   
    
    public CodeEnumTypeHandler(Class<CodeEnum> type) {    
        Objects.requireNonNull(type, "The type argument cannot be null");  
        Map<Integer, CodeEnum> map = new HashMap<>();     
        CodeEnum[] enumConstants = type.getEnumConstants();    
        for (CodeEnum item : enumConstants) {       
            map.put(item.getCode(), item);     
        }     
        this.type = type;      
        this.cache = map;
    }   
    
    @Override 
    public CodeEnum getNullableResult(ResultSet rs, String columnName)         throws SQLException {     
        int code = rs.getInt(columnName);    
        if(rs.wasNull()) {        
            return null;     
        } else { 
            return cache.get(code); 
        } 
    } 
    
    @Override 
    public CodeEnum getNullableResult(ResultSet rs, int columnIndex)         throws SQLException { 
        int code = rs.getInt(columnIndex);  
        if(rs.wasNull()) {    
            return null;    
        } else {    
            return cache.get(code); 
        }
    }
    
    @Override
    public CodeEnum getNullableResult(CallableStatement cs, int columnIndex)         throws SQLException {   
        int code = cs.getInt(columnIndex);
        if(cs.wasNull()) {      
            return null;     
        } else {   
            return cache.get(code); 
        } 
    } 
    
    @Override   
    public void setNonNullParameter(PreparedStatement ps, int i,                                    CodeEnum enumObj, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, enumObj.getCode());
    }
}
```

在mybatis.xml文件中配置：

```xml
<configuration>    
    <typeHandlers>
        <typeHandler handler="xxx.typehandler.CodeEnumTypeHandler"                javaType="yyy.domain.SexEnum"/>   
    </typeHandlers>
</configuration>
```