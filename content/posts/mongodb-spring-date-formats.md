+++
date = '2025-02-20T12:00:00+00:00'
draft = false
title = 'MongoDB and the Java DateTime API'
summary = "A quick guide on how avoid `java.util.Date`"
toc = true
readTime = true
autonumber = true
math = true
tags = ["java", "date", "mongodb", "spring"]
showTags = false
hideBackToTop = false
+++

## Date Handling in the MongoDB Driver
I will provide a bit of context before to explain the problem and our solution. You can directly jump to the fix [here](#the-fix).

### Background
When using the MongoDB in Java the driver allows for easy casting to the target object class. E.g. a BSON entry like:
```json
{
  "name": "foo",
  "time": "2020-01-01T12:00:00"
}
```
could be directly read as the POJO
```java
class Example {
    String name;
    Date time;
}
```
Now every experienced Java developer might cringe at seeing the type `Date` here. Java 8 introduced JSR-310 [^1] bringing the **Date and Time API** to developers.
Hence the POJO should rather look something like this:
```java
class Example {
    String name;
    LocalDateTime time;
}
```
The MongoDB driver supports JSR-310 since version 3.7 [^2] and if the POJO is declared in this way the driver should instantiate the time correctly (this is often done with automatic POJO codecs configured by frameworks like Spring Boot).

### The Issue
What happens when the type `LocalDateTime` is not enforced by the POJO? Let's consider setting the type to a generic `Object` (this might be useful if the type in the document might change or for something like a dynamic `Map<String, Object>` key-value map).
When instantiating the above BSON document to the POJO
```
class Example {
    String name;
    Object time;
}
```
the MongoDB driver with instantiate a `java.util.Date` object. MongoDB's default handling of `BSONType.DATE_TIME` is still `java.util.Date`, even if the driver supports JSR-310.

### Root cause
The origin of this issue is how the Java MongoDB driver deals with codecs. Documents get parsed as subparts depending on which codecs fits.
For each field the a codec is selected that fits best. The relevant class handles codec selection for BSON documents is the [`DocumentCodecProvider`](https://github.com/mongodb/mongo-java-driver/blob/main/bson/src/main/org/bson/codecs/DocumentCodecProvider.java). It has a function `get(Class<T> clazz, CodecRegistry)` which determines the codecs that will be used for the encoding/decoding.
Naturally, MongoDB has a default configuration of how BSON types should be cast to Java types (e.g. `BSONType.DOUBLE` to `java.lang.Double`).

Since our Java declaration with type `Object` doesn't enforce any type restrictions, MongoDB will refer to the defaults provided through the `BsonTypeClassMap.DEFAULT_BSON_TYPE_CLASS_MAP` ([here](https://github.com/mongodb/mongo-java-driver/blob/fb3f30b79aecfa5b17404c33a29fb1f5fb6f4ffb/bson/src/main/org/bson/codecs/BsonTypeClassMap.java#L112)). This is where we find the culprit: `BsonType.DATE_TIME` defaults to `Date.class`.
So to fix our problem we will have to tell MongoDB to use a different default codec and override the `CodecRegistry` [^3].

## The Fix
To fix the issue one just has to add a new class map. Below is an example on how to configure the MongoDB driver by adding a `MongoConfig` configuration class in Spring Boot.

```java
import com.mongodb.MongoClientSettings;
import java.time.LocalDateTime;
import java.util.Map;
import org.bson.BsonType;
import org.bson.codecs.BsonTypeClassMap;
import org.bson.codecs.DocumentCodecProvider;
import org.bson.codecs.configuration.CodecRegistries;
import org.bson.codecs.pojo.PojoCodecProvider;
import org.springframework.boot.autoconfigure.mongo.MongoClientSettingsBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**  
 * Configuration class for MongoDB.
 */
@Configuration  
public class MongoConfig {  
    /**  
     * Custom {@link MongoClientSettings} to decode {@link BsonType#DATE_TIME} to {@link LocalDateTime}
     * @return the customized settings builder
     */    
    @Bean  
    public MongoClientSettingsBuilderCustomizer mongoClientSettingsBuilderCustomizer() {
        BsonTypeClassMap classMap = new BsonTypeClassMap(Map.of(BsonType.DATE_TIME, LocalDateTime.class));  
        return builder -> builder.codecRegistry(CodecRegistries.fromRegistries(  
            CodecRegistries.fromProviders(new DocumentCodecProvider(classMap)),                // (1)
            MongoClientSettings.getDefaultCodecRegistry(),                                     // (2)
            CodecRegistries.fromProviders(PojoCodecProvider.builder().automatic(true).build()) // (3)
        ));  
    }
}
```

The most important element here is the `classMap`.
This map overrides the previously mentioned `DEFAULT_BSON_TYPE_CLASS_MAP` and enforces general use of the `LocalDateTimeCodec` class and even when casting to `Object`. MongoDB will always prefer to instantiate times as `LocalDateTime`.
The three codec registries are responsible for:
  1. overriding the default date handling
  2. providing a fall-back to the standard codecs
  3. providing a codec when casting to custom Java POJOs

Once setup the Spring bean injection will make sure MongoDB is appropriately configured. Configuring codecs for the MongoDB driver is also explained [here](https://www.mongodb.com/docs/drivers/java/sync/current/data-formats/codecs/).

[^1]: https://jcp.org/en/jsr/detail?id=310
[^2]: https://mongodb.github.io/mongo-java-driver/3.7/whats-new/#jsr-310-instant-localdate-localdatetime-support
[^3]: https://www.mongodb.com/docs/drivers/java/sync/current/faq/#can-i-serialize-a-java.util.date-as-a-string-in-format-yyyy-mm-dd-
