# Spring Boot Dual WAR/Executable JAR Build
> This is a basic scaffolding project for Spring Boot/Maven peojects which builds both a WAR and a JAR using the same code. 
> NOTE: Although outlined in this README, the JNDI lookup feature has only been implicitly tested. If the implementation 
  yields any errors, please let me know or fork/PR. This project has been tested with Java 8 and Tomcat 8.5 (both embedded and standalone) 

## Using Maven profiles, this project defines two build types:
    
    1. Using Maven profile dev, this would build and run a Spring Boot app 
    2. Using Maven profile release, this would build and run a standard WAR to be deployed on an app server (i.e. Tomcat) 

## Additional Features:
* A basic hello world service and unit test, referenced from the Spring Boot docs - https://spring.io/guides/gs/spring-boot/
Please take note of the different context URLs:
    * URL (for both JAR and WAR deployments): http://localhost:8080/springboot_war_and_jar_scaffold/greet/greeting
* Log4j2 - configured with some defaults (which is a lot of logging). Modify log4j2.xml under src/resources if you'd 
like different logging configuration
* Setup to use environment property files. To use this feature, create a application-{activeProfile}.properties file 
under src/resources.

## Usage: 

The two configured Maven commands. Note, {activeProfile} must match the naming convention of the property files 
    
1. Executable JAR - the 'dev' profile is active by default so no need to pass in a Maven profile  
    ```
        mvn clean install spring-boot:run -Dspring.profiles.active={activeProfile}
    ```

2. WAR  
    ```
        mvn clean install -P release -Dspring.profiles.active={activeProfile}
    ```

#### Tomcat Setup
* Under %TOMCAT_HOME%/conf, modify catalina.properties ton include the following line

    ``` sbtshell
        spring.profiles.active={activeProfile}
    ```
    
#### JNDI Setup

* To register a DataSource to use JNDI, add the following in Application.java. This is used for consistency between the 
two builds. It registers the DataSource as a Spring bean for bean management. I have not specified a destroyMethod 
since it should persists through the application's lifetime. I set the profile to all but test since I mock my unit 
tests (I connect externally only when performing integration tests). To use a DataSource during testing, just remove that line.

```java
 /**
 * Registers a DataSource as a JNDI lookup (opposed to any other method of DataSource defining Spring boot offers).
 * Used for consistency since JNDI is usually configured for DataSources in a standalone Tomcat.
 */
@Bean(destroyMethod = "")
@Profile("!test")
public DataSource jndiDataSource() throws NamingException {
    JndiObjectFactoryBean jndiFactoryBean = new JndiObjectFactoryBean();
    jndiFactoryBean.setJndiName("java:comp/env/" + jndiName);
    jndiFactoryBean.setProxyInterface(DataSource.class);
    jndiFactoryBean.setLookupOnStartup(true);
    jndiFactoryBean.afterPropertiesSet();
    return (DataSource) jndiFactoryBean.getObject();
}
```

Also, to ensure compatability, you'll want to modify application.properties in src/resources and add this to the T
omcatEmbeddedServletContainerFactory

```java
public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
    return new TomcatEmbeddedServletContainerFactory() {

        @Override
        protected TomcatEmbeddedServletContainer getTomcatEmbeddedServletContainer(Tomcat tomcat) {
            tomcat.enableNaming();
            return super.getTomcatEmbeddedServletContainer(tomcat);
        }

        @Override
        protected void postProcessContext(Context context) {
            ContextResource contextResource = new ContextResource();
            contextResource.setName(jndiName);
            contextResource.setAuth("Container");
            contextResource.setType("javax.sql.DataSource");
            contextResource.setProperty("url", url);
            contextResource.setProperty("username", username);
            contextResource.setProperty("password", password);
            contextResource.setProperty("initialSize", initialSize);
            contextResource.setProperty("maxWaitMillis", maxWait);
            contextResource.setProperty("maxTotal", maxActive);
            contextResource.setProperty("maxIdle", maxIdle);
            contextResource.setProperty("maxAge", maxAge);
            contextResource.setProperty("testOnBorrow", testOnBorrow);
            contextResource.setProperty("validationQuery", validationQuery);
            context.getNamingResources().addResource(contextResource);

        }
    }
}
```