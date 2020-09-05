---
layout: post
title:  Handling failures in Kafka streams
date:   2020-08-30 15:10:56 +0900
categories: kafka stream microservice
---

For a stream to be stable, resilient and reliable it is important that it handle failures gracefully. 
 Kafka streams failures are largely centered around the exceptions that occur during **deserialization** and  exceptions that occur during **interaction with broker**.

In this article I will explain how we can handle most of the fatal errors in kafka stream and avoid the application to shutdown unexpectedly.


## The failures

The data in a source topic of stream can have corrupt records for instance there can be records that doesn't follow the schema causing error on deserialization or there can be oversized records exceeding the `max.message.bytes` set for the topic its writing to, any occurence of such data can halt the stream and can cause serious disruption in the service as well data loss if not handled properly. 

### Deserialization exception
While consuming if a record doesn't follow the desired schema, it will throw deserialization exception, to handle it we setup a custom `DeserializationExceptionHandler`,  the defected record can be simply logged somewhere and the stream can continue without failing. 


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
These exceptions occur while the application interacts with the kafka broker. For example after processing a record the stream processor forwards it to a topic but the record size exceeds the `max.message.bytes`  set for the topic. Thus it will throw a `RecordTooLargeException`  by default the stream will fail, with a custom `ProductionExceptionHandler` we can handle the exception, and avoid stream failure.

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


### Other Runtime Failures

Apart from the above scenarios we can have Runtime failures like `IndexOutOfBoundsException` which comes up due to the processor-logic/data-issue or [`NoHostAvailableException`] [exception] which occurs while interacting with a database or a third party service.

These can be handled by simply using a try-catch in the processor code: 
```java 
try {  
    // code 
}  
catch (NoHostAvailableException e){  
  //handle, retry, notify..
}
```
All the failed records can be forwarded to some other topic for further analysis and reprocessing.

### When no option is left!

Sometimes we are  left with NO option if any uncaught exception occurs, all we can do is wait for the stream thread to die, but before it completely die we would trigger some code to notify that something wrong has occured in the stream. 
 `setUncaughtExceptionHandler` comes handy in such situation.

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
