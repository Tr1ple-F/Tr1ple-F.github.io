+++
date = '2025-02-20T12:00:00+00:00'
draft = false
title = 'Spring Boot, MongoDB and the Java DateTime API'
summary = "A quick fix guide on how avoid `java.util.Date`"
toc = true
readTime = true
autonumber = true
math = true
tags = ["java", "date", "mongodb", "spring"]
showTags = false
hideBackToTop = false
+++

## What & Why?

### Background
In the techstack of my previous startup we used Java/Spring as our backend framework and MongoDB as database.
One of the reasons for choosing this stack was the security features provided by Spring (the headaches of which I will describe in a future post).
As for our database, we had a fairly flexible schema of documents we had to accommodate. 

Take these two example documents:
```json
{
  "name": "foo",
  "time": "2020-01-01T12:00:00",
  "params": {
    "foo": 1,
    "bar": "2024-01-01T12:00:00"
  }
}
{
  "name": "bar",
  "time": "2020-01-01T12:00:00",
  "params": {
    "foo": "string",
    "bar": "2021-01-01T12:00:00"
  }
}
```

We were constrained by
1. we couldn't fix the keys inside `params` from our application side
2. we couldn't fix the types of the keys ahead of time.

Hence our Java code looked something like this:
```java
class MyDocument {
    String name;
    LocalDateTime time;
    Map<String, Object> params;
}
```
where our backend code would rely on MongoDB to correctly serialize the `params` map which we would then process.

### The Issue
When serializing the `time` field, MongoDB would correctly parse the BSON representation to `LocalDateTime` (i.e. the JSR-310 **Java 8 Date/Time API**).
However, since MongoDB's default handling of `BSONType.DATE_TIME` is still `java.util.Date` our `params` map would contain objects of type `java.util.Date` which we would have to re-cast.
Naturally, this introduced many bugs and crashes on our end whenever we had some random `java.util.Date` object flying through our memory. Pretty much from the start of our project this bug bothered us
but we didn't manage to take care of it until almost 2.5 years later.

### MongoDB Codecs
The origin of this issue is how the Java MongoDB driver deals with codecs.
The MongoDB driver supports JSR-310 since version 3.7 [^1]; however, documents get parsed as subparts depending on which codecs fits.
For fields this can happen completely independently of each other which means two fields could use completely different codecs.
The relevant class handles codec selection for BSON docuemnts is the `DocumentCodecProvider`.
It has a function `get(Class<T> clazz, CodecRegistry)` which determines the codecs that will be used throughout the encoding/decoding.
Naturally, MongoDB has a default configuration of how BSON types should be cast to Java types (e.g. `BSONType.DOUBLE` to `java.lang.Double`).
Since our Java declaration of `params` doesn't enforce any type restrictions, MongoDB will refer to the defaults provided through the `BsonTypeClassMap.DEFAULT_BSON_TYPE_CLASS_MAP`.
[Here](https://github.com/mongodb/mongo-java-driver/blob/fb3f30b79aecfa5b17404c33a29fb1f5fb6f4ffb/bson/src/main/org/bson/codecs/BsonTypeClassMap.java#L112) we find the culprit: `BsonType.DATE_TIME` defaults to `Date.class`.
So to fix our problem we will have to tell MongoDB to use a different default codec and override the `CodecRegistry` [^2].

## The Fix
To fix the issue, we added a class named `MongoConfig` to our application which configures the MongoDB BSON codec mapping.

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
This map overrides the previously mentioned `DEFAULT_BSON_TYPE_CLASS_MAP` and enforces general use of the `LocalDateTimeCodec` class and even when casting to `Object` MongoDB will default to instantiate the object as `LocalDateTime`.
The three codec registries are responsible for (1) overriding the default date handling (2) refering otherwise to default codecs (3) providing a codec when casting to custom Java POJOs.
Once setup the Spring bean injection will make sure MongoDB is appropriately configured.

---

[^1]: https://mongodb.github.io/mongo-java-driver/3.7/whats-new/#jsr-310-instant-localdate-localdatetime-support
[^2]: https://www.mongodb.com/docs/drivers/java/sync/current/faq/#can-i-serialize-a-java.util.date-as-a-string-in-format-yyyy-mm-dd-
