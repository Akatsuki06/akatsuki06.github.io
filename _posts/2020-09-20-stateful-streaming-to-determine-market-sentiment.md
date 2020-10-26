---
layout: post
title:  Stateful streaming to analyse market sentiment
date:   2020-09-20 12:01:56 +0530
categories: kafka stream microservice
---

Notion of a state in a stream processing may sound odd as most of the stream processing would only need continuos flow of discrete events but sometimes we may require to extract some meaningful insights from the flowing data in a time bound frame.

> A perfect example is a stock market prices keep fluctuating, so in a 10 minute of time frame the prices go up and down multiple times, we can use prices of last 10 minutes to whether a stock is overbought or oversold! If a particular stock be overbought, implies that its a good candidate to be sold hence the market will pullback in near future. Market sentiment is price driven and time bound. Hence we will collect all prices in a time frame using kafka stream and will store the prices in state store, the market sentiment will be evaluated for the period and published to other topic from where it can be used by other applications.



## Topology

![Image](https://user-images.githubusercontent.com/16136908/93705864-9dab1300-fb3e-11ea-9ede-8e6342ea4038.jpg)


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
