# spring-boot-websphere-mq8-connect
Send message to IBM websphere mq8 using spring boot 
## Introduction
This git repository explain how to connect IBM websphere MQ8 from spring boot to send the message to queue.
##  Follow below steps to send message to IBM Websphere MQ from spring boot
### Step1:
Add the MQ details to `application.properties` file.

Example:
````
cdi.mq.queueManager= << queueManager >>
cdi.mq.host= << xx.xx.xx.xxx >>
cdi.mq.port= << port >>
cdi.mq.channel= << channel >>
cdi.mq.userid=MUSR_MQADMIN 
cdi.mq.password=password
````
> Note: Default IBM  websphere MQ user id `MUSR_MQADMIN  and password is `password`
    
### Step2:
Add the java class to read the properties.

````
@ConfigurationProperties(prefix = "cdi.mq")
@Data
public class MQProperties {
    String queueManager;
    String host;
    int port;
    String channel;
    String userid;
    String password;
}
````
### Step3:
Add the config class to create MQQueueConnectionFactory and jmsTemplate.

````
@Configuration
@Profile("non-jee")
public class AppConfig {

    @Autowired
    private MQProperties mqProperties;

    JmsTemplate jmsTemplate;

    @Bean
    public JmsTemplate jmsTemplate() {
        JmsTemplate jmsTemplate = new JmsTemplate(userCredentialsConnectionFactoryAdapter());
        return jmsTemplate;
    }

    @Bean
    public MQProperties mqProperties() {
        return new MQProperties();
    }
    @Bean
    @Primary
    public MQQueueConnectionFactory mqQueueConnectionFactory() {
        MQQueueConnectionFactory mqQueueConnectionFactory = new MQQueueConnectionFactory();
        mqQueueConnectionFactory.setHostName(mqProperties.getHost());
        try {
            mqQueueConnectionFactory.setTransportType(WMQConstants.WMQ_CM_CLIENT);
            mqQueueConnectionFactory.setCCSID(1381);
            mqQueueConnectionFactory.setChannel(mqProperties.getChannel());
            mqQueueConnectionFactory.setPort(mqProperties.getPort());
            mqQueueConnectionFactory.setMsgBatchSize(10000);
            mqQueueConnectionFactory.setQueueManager(mqProperties.getQueueManager());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return mqQueueConnectionFactory;
    }

    @Bean
    public UserCredentialsConnectionFactoryAdapter userCredentialsConnectionFactoryAdapter() {
        UserCredentialsConnectionFactoryAdapter userCredentialsConnectionFactoryAdapter = new UserCredentialsConnectionFactoryAdapter();
        userCredentialsConnectionFactoryAdapter.setUsername(mqProperties.getUserid());
        userCredentialsConnectionFactoryAdapter.setPassword(mqProperties.getPassword());
        userCredentialsConnectionFactoryAdapter.setTargetConnectionFactory(mqQueueConnectionFactory());
        return userCredentialsConnectionFactoryAdapter;
    }

}
````
### Step4:

Add the message sender class to send message to MQ.

````
@Component
public class MessageSender {
    private static final String SIMPLE_QUEUE = "MQ_NAME";

    @Autowired
    private JmsTemplate jmsTemplate;

    public void send(final Object Object) {
        jmsTemplate.convertAndSend(SIMPLE_QUEUE, Object);
    }
}
````
### Step5:
Add the com.ibm.mq.allclient jar in pom.xml.
> Note: com.ibm.mq.allclient jar is available in the IBM Websphere MQ installation directory [`C:\Program Files\ibm\WebSphere MQ\java\lib\ `]

````
<dependency>
    <groupId>com.ibm.mq</groupId>
    <artifactId>allclient</artifactId>
    <version>8.0.0.2</version>
</dependency>
````

##  Follow below steps to send message to IBM Websphere MQ from spring boot app deployed in IBM websphere liberty.
### Step1:
Add the following features in server.xml
````
  <feature>localConnector-1.0</feature>
  <feature>jms-2.0</feature>
  <feature>jca-1.7</feature>
  <feature>webProfile-7.0</feature>
  <feature>wmqJmsClient-2.0</feature>
  <feature>jndi-1.0</feature>	
````

### Step2:
Add the following rar file to the websphere liberty server root directory
> Note: > Note: wmq.jmsra.rar is available in the IBM Websphere MQ installation directory.
````
wmq.jmsra.rar
````
### Step 3:
Add the following mq connection details in server.xml

````
<variable name="wmqJmsClient.rar.location" value="${server.config.dir}/wmq.jmsra.rar"/>

<jmsConnectionFactory jndiName="jms/CustomerCF" connectionManagerRef="ConMgr">
  <properties.wmqJms 
  hostName="<< xx.xx.xx.xxx >>" 
  port="<< port >>"
  channel="<< channel >>"
  queueManager="<< queueManager >>"
  userName="MUSR_MQADMIN"
  password="password"
  />
</jmsConnectionFactory>

<connectionManager id="ConMgr" maxPoolSize="10"/>

<jmsQueue id="PartyQueue" jndiName="jms/CustomerQueue">
    <properties.wmqJms baseQueueName="MQ_NAME" baseQueueManagerName="<< queueManager >>"/>
</jmsQueue> 

<jmsActivationSpec id="PartyJmsActivationSpec">
    <properties.wmqJms destinationRef="CustomerQueue" 
                 destinationType="javax.jms.Queue" 
                 queueManager="<< queueManager >>"/>
</jmsActivationSpec>
	
````
### Step 4:
Add the resource reference in web.xml

````
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
  <display-name>Customer Rest</display-name>

  <resource-ref>
    <res-ref-name>jms/CustomerCF</res-ref-name>
    <res-type>javax.jms.ConnectionFactory</res-type>
    <res-auth>Container</res-auth>
    <res-sharing-scope>Shareable</res-sharing-scope>
  </resource-ref>
</web-app>
````
### Step 5:
Add the MQ jndi to `application.properties` file.

Example:
````
spring.jms.jndi-name=jms/CustomerCF
````

We can use both approach like Spring boot run in in-build tomcat and also deploy in websphere liberty. Just split the code using spring profiles.