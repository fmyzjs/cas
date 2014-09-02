layout: default
title: CAS - Service Management
---

# Service Management
The CAS service management facility allows CAS server administrators to declare and configure which services
(CAS clients) may make use of CAS in which ways. The core component of the service management facility is the
service registry, provided by the `ServiceRegistryDao` component, that stores one or more registered services
containing metadata that drives a number of CAS behaviors:

* Authorized services - Control which services may participate in a CAS SSO session.
* Forced authentication - Provides administrative control for forced authentication.
* Attribute release - Provide user details to services for authorization and personalization.
* Proxy control - Further restrict authorized services by granting/denying proxy authentication capability.
* Theme control - Define alternate CAS themese to be used for particular services.

The service management console is a Web application that may be deployed along side CAS that provides a GUI
to manage service registry data.


## Demo
The Service Management web application is available for demo at [https://jasigcasmgmt.herokuapp.com](https://jasigcasmgmt.herokuapp.com)


## Considerations
It is not required to use the service management facility explicitly. CAS ships with a default configuration that is
suitable for deployments that do not need or want to leverage the capabilities above. The default configuration allows
any service contacting CAS over https/imaps to use CAS and receive any attribute configured by an `IPersonAttributeDao`
bean.


A premier consideration around service management is whether to leverage the user interface. If the service management
console is used, then a `ServiceRegistryDao` that provides durable storage (e.g. `JpaServiceRegistryDaoImpl`) must be
used to preserve state across restarts.

It is perfectly acceptable to avoid the service management console Web application for managing registered service data.
In fact, configuration-driven methods (e.g. XML, JSON) may be preferable in environments where strict configuration
management controls are required.


## Registered Services

Registered services present the following metadata:

| Field         					| Description 
|-----------------------------------+--------------------------------------------------------------------------------+
| `id`     							| Required unique identifier. In most cases this is managed automatically by the `ServiceRegistryDao`.
| `name`        					| Required name (255 characters or less).      
| `description`						| Optional free-text description of the service. (255 characters or less)   
| `serviceId`        				| Required [Ant pattern](http://ant.apache.org/manual/dirtasks.html#patterns) or [regular expression](http://docs.oracle.com/javase/tutorial/essential/regex/) describing a logical service. A logical service defines one or more URLs where a service or services are located. The definition of the url pattern must be **done carefully** because it can open security breaches. For example, using Ant pattern, if you define the following service : `http://example.*/myService` to match `http://example.com/myService` and `http://example.fr/myService`, it's a bad idea as it can be tricked by `http://example.hostattacker.com/myService`. The best way to proceed is to define the more precise url patterns.
| `theme`        					| Optional [Spring theme](http://static.springsource.org/spring/docs/3.2.x/spring-framework-reference/html/mvc.html#mvc-themeresolver) that may be used to customize the CAS UI when the service requests a ticket. See [this guide](User-Interface-Customization.html) for more details on attribute release and filters.
| `enabled`        					| Flag to toggle whether the entry is active; a disabled entry produces behavior equivalent to a non-existent entry.
| `ssoEnabled`       				| Set to false to force users to authenticate to the service regardless of protocol flags (e.g. `renew=true`). This flag provides some support for centralized application of security policy.
| `anonymousAccess`        			| Set to true to provide an opaque identifier for the username instead of the principal ID. The default behavior (false) is to release the principal ID. The identifier conforms to the requirements of the [eduPersonTargetedID](http://www.incommon.org/federation/attributesummary.html#eduPersonTargetedID) attribute.
| `proxyPolicy`        				| Determines whether the service is able to proxy authentication, not whether the service accepts proxy authentication. 
| `evaluationOrder`        			| Required value that determines relative order of evaluation of registered services. This flag is particularly important in cases where two service URL expressions cover the same services; evaluation order determines which registration is evaluated first.      
| `usernameAttribute`        		| Name of the attribute that would identify the principal for this service definition only. The attribute need not be configured in the release policy, but must only be resolvable by the attribute repository.
| `requiredHandlers`        		| Set of authentication handler names that must successfully authenticate credentials in order to access the service.
| `attributeReleasePolicy`        	| The policy that describes the set of attributes allows to be released to the application, as well as any other filtering logic needed to weed some out. See [this guide](../integration/Attribute-Release.html) for more details on attribute release and filters.
| `logoutType`        				| Defines how this service should be treated once the logout protocol is initiated. Acceptable values are `LogoutType.BACK_CHANNEL`, `LogoutType.FRONT_CHANNEL` or `LogoutType.NONE`. See [this guide](Logout-Single-Signout.html) for more details on logout. 

###Configure Proxy Authentication Policy
Each registered application in the registry may be assigned a proxy policy to determine whether the service is allowed for proxy authentication. This means that a PGT will not be issued to a service unless the proxy policy is configured to allow it. Additionally, the policy could also define which endpoint urls are in fact allowed to receive the PGT. 

Note that by default, the proxy authentication is disallowed for all applications.

####Components 

#####`RefuseRegisteredServiceProxyPolicy`
Disallows proxy authentication for a service. This is default policy and need not be configured explicitly.

#####`RegexMatchingRegisteredServiceProxyPolicy`
A proxy policy that only allows proxying to pgt urls that match the specified regex pattern.

{% highlight xml %}
<bean class="org.jasig.cas.services.RegexRegisteredService"
         p:id="10000001" p:name="HTTP and IMAP"
         p:description="Allows HTTP(S) and IMAP(S) protocols"
         p:serviceId="^(https?|imaps?)://.*" p:evaluationOrder="10000001">
	<property name="proxyPolicy">
		<bean class="org.jasig.cas.services.RegexMatchingRegisteredServiceProxyPolicy"
			c:pgtUrlPattern="^https?://.*" />
	</property>
</bean>
{% endhighlight %}

## Persisting Registered Service Data

######`InMemoryServiceRegistryDaoImpl`
CAS uses in-memory services management by default, with the registry seeded from registration beans wired via Spring.

{% highlight xml %}
<bean id="serviceRegistryDao"
      class="org.jasig.cas.services.InMemoryServiceRegistryDaoImpl"
      p:registeredServices-ref="registeredServicesList" />

<util:list id="registeredServicesList">
    <bean class="org.jasig.cas.services.RegexRegisteredService"
          p:id="1"
          p:name="HTTPS and IMAPS services on example.com"
          p:serviceId="^(https|imaps)://([A-Za-z0-9_-]+\.)*example\.com/.*"
          p:evaluationOrder="0" />
</util:list>

{% endhighlight %}

This component is _NOT_ suitable for use with the service management console since it does not persist data.
On the other hand, it is perfectly acceptable for deployments where the XML configuration is authoritative for
service registry data and the UI will not be used.


######`LdapServiceRegistryDao`
Service registry implementation which stores the services in a LDAP Directory. Uses an instance of `LdapRegisteredServiceMapper`, that by default is `DefaultLdapServiceMapper` in order to configure settings for retrieval, search and persistence of service definitions. By default, entries are assigned the `objectclass` `casRegisteredService` attribute and are looked up by the `uid` attribute.

{% highlight xml %}

<context:component-scan base-package="org.jasig.cas" />

<bean id="serviceRegistryDao"
      class="org.jasig.cas.adaptors.ldap.services.LdapServiceRegistryDao"
      p:connectionFactory-ref="pooledLdapConnectionFactory"
      p:searchRequest-ref="searchRequest"
      p:ldapServiceMapper-ref="ldapMapper" />

<bean id="ldapMapper"
      class="org.jasig.cas.adaptors.ldap.services.DefaultLdapServiceMapper"/>
{% endhighlight %}

Note that you will need to add the `context` namespace and schema location to the `beans` declaration of the file.

<p/>

######`DefaultLdapServiceMapper`
The default mapper has support for the following items:

| Field         					| Default Value 
|-----------------------------------+--------------------------------------------------------------------------------+
| `objectClass`     				| casRegisteredService
| `serviceIdAttribute`     			| casServiceUrlPattern
| `idAttribute`     				| uid
| `serviceDescriptionAttribute`     | description
| `serviceNameAttribute`     		| cn
| `serviceEnabledAttribute`     	| casServiceEnabled
| `serviceSsoEnabledAttribute`     	| casServiceSsoEnabled
| `serviceAnonymousAccessAttribute` | casServiceAnonymousAccess
| `serviceProxyPolicyAttribute`     | casServiceProxyPolicy
| `serviceThemeAttribute`     		| casServiceTheme
| `usernameAttribute`     			| casUsernameAttribute
| `attributeReleasePolicyAttribute` | casAttributeReleasePolicy
| `evaluationOrderAttribute`     	| casEvaluationOrder
| `requiredHandlersAttribute`     	| casRequiredHandlers


######`JpaServiceRegistryDaoImpl`
Stores registered service data in a database; the preferred choice when using the service management console.
The following configuration template may be applied to `deployerConfigContext.xml` to provide for persistent
registered service storage. The configuration assumes a `dataSource` bean is defined in the context.

{% highlight xml %}
<tx:annotation-driven transaction-manager-ref="transactionManager" />

<bean id="factoryBean"
      class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean"
      p:dataSource-ref="dataSource"
      p:jpaVendorAdapter-ref="jpaVendorAdapter"
      p:packagesToScan-ref="packagesToScan">
    <property name="jpaProperties">
      <props>
        <prop key="hibernate.dialect">${database.dialect}</prop>
        <prop key="hibernate.hbm2ddl.auto">update</prop>
        <prop key="hibernate.jdbc.batch_size">${database.batchSize}</prop>
      </props>
    </property>
</bean>

<bean id="jpaVendorAdapter"
      class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter"
      p:generateDdl="true"
      p:showSql="true" />

<bean id="serviceRegistryDao"
      class="org.jasig.cas.services.JpaServiceRegistryDaoImpl" />

<bean id="transactionManager"
      class="org.springframework.orm.jpa.JpaTransactionManager"
      p:entityManagerFactory-ref="factoryBean" />

<!--
   | Injects EntityManager/Factory instances into beans with
   | @PersistenceUnit and @PersistenceContext
-->
<bean class="org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor" />

<!--
   Configuration via JNDI
-->
<bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean"
    p:jndiName="java:comp/env/jdbc/cas-source" />   
{% endhighlight %}

If you prefer a direct connection to the database, here's a sample configuration of the `dataSource`:

{% highlight xml %}
 <bean
        id="dataSource"
        class="com.mchange.v2.c3p0.ComboPooledDataSource"
        p:driverClassName="org.hsqldb.jdbcDriver"
        p:jdbcUrl-ref="database"
        p:password=""
        p:username="sa" />
{% endhighlight %}

The data source will need to be modified for your particular database (i.e. Oracle, MySQL, etc.), but the name `dataSource` should be preserved. Here is a MYSQL sample:

{% highlight xml %}
<bean
        id="dataSource"
        class="org.apache.commons.dbcp.BasicDataSource"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/test?autoReconnect=true"
        p:password=""
        p:username="sa" />
{% endhighlight %}

You will also need to change the property `hibernate.dialect` in adequacy with your database in `cas.properties` and `deployerConfigContext.xml`. 

For example, for MYSQL the setting would be:

In `cas.properties`:

{% highlight bash %}
database.hibernate.dialect=org.hibernate.dialect.MySQLDialect
{% endhighlight %}

In `deployerConfigContext.xml`:

{% highlight xml %}
<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
{% endhighlight %}

You will also need to ensure that the xml configuration file contains the `tx` namespace:

{% highlight xml %}
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">
{% endhighlight %}

Finally, when adding a new source new dependencies may be required on Hibernate, commons-dbcp. Be sure to add those to your `pom.xml`. Below is a sample configuration for MYSQL. Be sure to adjust the version elements for the appropriate version number.

{% highlight xml %}
<dependency>
    <groupId>commons-dbcp</groupId>
    <artifactId>commons-dbcp</artifactId>
    <version>${commons.dbcp.version}</version>
    <scope>runtime</scope>
</dependency>
 
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>${hibernate.version}</version>
    <scope>compile</scope>
</dependency>
 
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-entitymanager</artifactId>
    <version>${hibernate.entitymgmr.version}</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>${mysql.connector.version}</version>
</dependency>

{% endhighlight %}

## Installing the Services Management Webapp

The services management webapp is no more part of the CAS server and is a standalone web application: `cas-management-webapp`.

Nonetheless, one must keep in mind that both applications (the CAS server and the services management webapp) share the _same_ configuration for the CAS services:

* the management webapp is used to add/edit/delete all the CAS services
* the CAS server loads/relies on all these defined CAS services to process all incoming requests.

You can install the services management webapp in your favourite applications server, there is no restriction.
Though, you need at first to configure it according to your environment. Towards that goal, the best way to proceed is to create your own services management webapp using a [Maven overlay](http://maven.apache.org/plugins/maven-war-plugin/overlays.html) based on the CAS services management webapp:

{% highlight xml %}
<dependency>
  <groupId>org.jasig.cas</groupId>
  <artifactId>cas-management-webapp</artifactId>
  <version>${cas.version}</version>
  <type>war</type>
  <scope>runtime</scope>
</dependency>
{% endhighlight %}


### Authentication method

By default, the `cas-management-webapp` is configured to authenticate against a CAS server. We assume that it's the case in this documentation. However, you could change the authentication method by overriding the `WEB-INF/spring-configuration/securityContext.xml` file.


###Securing Access and Authorization
Access to the management webapp is controlled via Spring Security. Rules are defined in the `/cas-management-webapp/src/main/webapp/WEB-INF/managementConfigContext.xml` file.


####Static List of Users
By default, access is limited to a static list of users whose credentials may be specified in a `user-details.properties` file that should be available on the runtime classpath. 

{% highlight xml %}
<sec:user-service id="userDetailsService" 
   properties="${user.details.file.location:classpath:user-details.properties}" />
{% endhighlight %}

You can change the location of this file, by uncommenting the following key in your `cas-management.properties` file:

{% highlight bash %}
##
# User details file location that contains list of users
# who are allowed access to the management webapp:
# 
# user.details.file.location = classpath:user-details.properties
{% endhighlight %}

The format of the file should be as such:

{% highlight bash %}
# The syntax of each entry should be in the form of:
# 
# username=password,grantedAuthority[,grantedAuthority][,enabled|disabled]

# Example:
# casuser=notused,ROLE_ADMIN
{% endhighlight %}


####LDAP-managed List of Users
If you wish allow access to the services management application via an LDAP group/server, open up the `deployerConfigContext` file of the management web application and adjust for the following:

{% highlight xml %}
<sec:ldap-server id="ldapServer" url="ldap://myserver:13060/"
                 manager-dn="cn=adminusername,cn=Users,dc=london-scottish,dc=com"
                 manager-password="mypassword" />
<sec:ldap-user-service id="userDetailsService" server-ref="ldapServer"
            group-search-base="cn=Groups,dc=mycompany,dc=com" group-role-attribute="cn"
            group-search-filter="(uniquemember={0})"
            user-search-base="cn=Users,dc=mycompany,dc=com"
            user-search-filter="(uid={0})"/>
{% endhighlight %}

You will also need to ensure that the `spring-security-ldap` dependency is available to your build at runtime:

{% highlight xml %}
<dependency>
   <groupId>org.springframework.security</groupId>
   <artifactId>spring-security-ldap</artifactId>
   <version>${spring.security.ldap.version}</version>
   <exclusions>
     <exclusion>
             <groupId>org.springframework</groupId>
             <artifactId>spring-aop</artifactId>
     </exclusion>
     <exclusion>
             <groupId>org.springframework</groupId>
             <artifactId>spring-tx</artifactId>
     </exclusion>
     <exclusion>
             <groupId>org.springframework</groupId>
             <artifactId>spring-beans</artifactId>
     </exclusion>
     <exclusion>
             <groupId>org.springframework</groupId>
             <artifactId>spring-context</artifactId>
     </exclusion>
     <exclusion>
             <groupId>org.springframework</groupId>
             <artifactId>spring-core</artifactId>
     </exclusion>
   </exclusions>
</dependency>
{% endhighlight %}


### Urls configuration

The urls configuration of the CAS server and management applications can be done by overriding the default `WEB-INF/cas-management.properties` file:

    # CAS
    cas.host=http://localhost:8080
    cas.prefix=${cas.host}/cas
    cas.securityContext.casProcessingFilterEntryPoint.loginUrl=${cas.prefix}/login
    cas.securityContext.ticketValidator.casServerUrlPrefix=${cas.prefix}
    # Management
    cas-management.host=${cas.host}
    cas-management.prefix=${cas-management.host}/cas-management
    cas-management.securityContext.serviceProperties.service=${cas-management.prefix}/j_spring_cas_security_check
    cas-management.securityContext.serviceProperties.adminRoles=ROLE_ADMIN

When authenticating against a CAS server, the services management webapp will be processed as a regular CAS service and thus, needs to be defined in the services registry (of the CAS server).


### Services registry

You also need to define the *common* services registry by overriding the `WEB-INF/managementConfigContext.xml` file and set the appropriate `serviceRegistryDao` (see above: *Persisting Registered Service Data*). It should be the same configuration you already use in your CAS server (in the `WEB-INF/deployerConfigContext.xml` file).


### UI

The services management webapp is pretty simple to use:

* use the "Manage Services" link to see the list of all CAS services
* click the "Add New Service" link to add a new CAS service
* click the "edit" link with the pen image (on the right of a CAS service definition) to edit a specific CAS service
* click the "delete" link with the trash image (on the right of a CAS service definition) to delete a specific CAS service (after a confirmation alert).
* The app takes advantage of the Infusion Javascript framework in order to add drag&drop functionality onto the screen.
