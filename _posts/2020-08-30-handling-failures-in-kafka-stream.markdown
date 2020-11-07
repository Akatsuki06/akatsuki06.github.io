---
layout: post
title:  Handling failures in Kafka streams
date:   2020-08-30 15:10:56 +0900
categories: kafka stream microservice
---

For a Kafka stream to be stable, resilient and reliable it is important that it handle failures gracefully. 
Occurrence of failures can halt the stream and can cause serious disruption in the service.

Even if exceptions occurring within the code are handled there is a fair possibility of stream failure due to inconsistency in schema or due to failure on client-broker interaction, which can lead to unexpected shutdown of the application.

This article talks about various kind of exception occurring in kafka stream and how to handle them.



### Exceptions within the code

Exceptions which are thrown by the processor code such as arithmetic exceptions, parsing exceptions, or timeout exception on database call. These can be handled by simply putting a try-catch block around the piece of code which throws the exception.

```java 
try {  
    user = db.getUser(id) 
}  
catch (ReadTimeoutException e){  
  //handle, retry, notify..
}
```

### Deserialization exception

These exception are thrown by kafka and can not be handled by a try-catch in our code.
The data present in a kafka topic may have a different schema then what the stream consumer expects. This throws a deserialization exception when consumed by the consumer.
To handle such failures kafka provides `DeserializationExceptionHandler`. As these failures occur due to inconsistent data in topic, they can be simply logged and the stream can continue without failing.

```java
public class CustomDeserializationExceptionHandler implements DeserializationExceptionHandler {  
  
  @Override  
  public DeserializationHandlerResponse handle(ProcessorContext context, ConsumerRecord<byte[], byte[]> record, 
  									Exception exception) {  
        //logic to deal with the corrupt record  
  	return DeserializationHandlerResponse.CONTINUE;  
    }  
  
  @Override  
  public void configure(Map<String, ?> map) {  
        //can read any property set in the stream here  
  }  
}

// in the streams config
streamConfig.put("default.deserialization.exception.handler", CustomDeserializationExceptionHandler.class);

```

### Production exceptions

Any exception that occur during kafka broker and client interaction is a production exception. An example is `RecordTooLargeException`.
If the kafka stream writes a record to a topic but the record size exceeds the largest message size allowed by the broker(`max.message.bytes`), it will throw a `RecordTooLargeException`. We can handle the exception using a custom `ProductionExceptionHandler`.

```java 
public class CustomProductionExceptionHandler implements ProductionExceptionHandler {  
  @Override  
  public ProductionExceptionHandlerResponse handle(ProducerRecord<byte[], byte[]> record,
  								 Exception exception) {  
          
        if (exception instanceof RecordTooLargeException){  
            // code to deal with large record..  
 		 return ProductionExceptionHandlerResponse.CONTINUE;  
        }  
          //other conditional checks
          
        return ProductionExceptionHandlerResponse.FAIL;  
    }  
  
  @Override  
  public void configure(Map<String, ?> map) {  
  
    }  
}

//in the streams config
streamConfig.put("default.production.exception.handler", CustomProductionExceptionHandler.class);

```

### Uncaught exceptions

If any uncaught exception occurs in a stream thread and kills the thread. The `setUncaughtExceptionHandler` method provides a way to trigger the last code when the thread abruptly terminates.

```java
kafkaStreams.setUncaughtExceptionHandler((thread,exception) ->{
    //code to be triggered
});

```

<br>

<hr/>

References:

- [Apache Kafka][apache-kafka]
- [Confluent docs][confluent-kafka]

[exception]: https://docs.datastax.com/en/drivers/java/2.0/com/datastax/driver/core/exceptions/NoHostAvailableException.html
[uber-reliable-reprocessing]: https://eng.uber.com/reliable-reprocessing/
[apache-kafka]: https://kafka.apache.org/
[confluent-kafka]: https://docs.confluent.io/current/index.html
