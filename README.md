
### Apache DS
    https://stackoverflow.com/questions/1560230/running-apache-ds-embedded-in-my-application
    http://directory.apache.org/
    
### https://docs.spring.io/spring-security/site/docs/3.0.x/reference/ldap.html

18. LDAP Authentication
Prev 	Part V. Additional Topics	 Next
LDAP Authentication
18.1 Overview
LDAP is often used by organizations as a central repository for user information and as an authentication service. It can also be used to store the role information for application users.

There are many different scenarios for how an LDAP server may be configured so Spring Security's LDAP provider is fully configurable. It uses separate strategy interfaces for authentication and role retrieval and provides default implementations which can be configured to handle a wide range of situations.

You should be familiar with LDAP before trying to use it with Spring Security. The following link provides a good introduction to the concepts involved and a guide to setting up a directory using the free LDAP server OpenLDAP: http://www.zytrax.com/books/ldap/. Some familiarity with the JNDI APIs used to access LDAP from Java may also be useful. We don't use any third-party LDAP libraries (Mozilla, JLDAP etc.) in the LDAP provider, but extensive use is made of Spring LDAP, so some familiarity with that project may be useful if you plan on adding your own customizations.

18.2 Using LDAP with Spring Security
LDAP authentication in Spring Security can be roughly divided into the following stages.

Obtaining the unique LDAP “Distinguished Name”, or DN, from the login name. This will often mean performing a search in the directory, unless the exact mapping of usernames to DNs is known in advance.

Authenticating the user, either by binding as that user or by performing a remote “compare” operation of the user's password against the password attribute in the directory entry for the DN.

Loading the list of authorities for the user.

The exception is when the LDAP directory is just being used to retrieve user information and authenticate against it locally. This may not be possible as directories are often set up with limited read access for attributes such as user passwords.

We will look at some configuration scenarios below. For full information on available configuration options, please consult the security namespace schema (information from which should be available in your XML editor).

18.3 Configuring an LDAP Server
The first thing you need to do is configure the server against which authentication should take place. This is done using the <ldap-server> element from the security namespace. This can be configured to point at an external LDAP server, using the url attribute:

  <ldap-server url="ldap://springframework.org:389/dc=springframework,dc=org" />

            
18.3.1 Using an Embedded Test Server
The <ldap-server> element can also be used to create an embedded server, which can be very useful for testing and demonstrations. In this case you use it without the url attribute:

  <ldap-server root="dc=springframework,dc=org"/>
 
    
Here we've specified that the root DIT of the directory should be “dc=springframework,dc=org”, which is the default. Used this way, the namespace parser will create an embedded Apache Directory server and scan the classpath for any LDIF files, which it will attempt to load into the server. You can customize this behaviour using the ldif attribute, which defines an LDIF resource to be loaded:

  <ldap-server ldif="classpath:users.ldif" />
        
This makes it a lot easier to get up and running with LDAP, since it can be inconvenient to work all the time with an external server. It also insulates the user from the complex bean configuration needed to wire up an Apache Directory server. Using plain Spring Beans the configuration would be much more cluttered. You must have the necessary Apache Directory dependency jars available for your application to use. These can be obtained from the LDAP sample application.

18.3.2 Using Bind Authentication
This is the most common LDAP authentication scenario.

  <ldap-authentication-provider user-dn-pattern="uid={0},ou=people"/>
                     
This simple example would obtain the DN for the user by substituting the user login name in the supplied pattern and attempting to bind as that user with the login password. This is OK if all your users are stored under a single node in the directory. If instead you wished to configure an LDAP search filter to locate the user, you could use the following:

  <ldap-authentication-provider user-search-filter="(uid={0})"
          user-search-base="ou=people"/>
                    
If used with the server definition above, this would perform a search under the DN ou=people,dc=springframework,dc=org using the value of the user-search-filter attribute as a filter. Again the user login name is substituted for the parameter in the filter name. If user-search-base isn't supplied, the search will be performed from the root.

18.3.3 Loading Authorities
How authorities are loaded from groups in the LDAP directory is controlled by the following attributes.

group-search-base. Defines the part of the directory tree under which group searches should be performed.

group-role-attribute. The attribute which contains the name of the authority defined by the group entry. Defaults to cn

group-search-filter. The filter which is used to search for group membership. The default is uniqueMember={0}, corresponding to the groupOfUniqueMembers LDAP class. In this case, the substituted parameter is the full distinguished name of the user. The parameter {1} can be used if you want to filter on the login name.

So if we used the following configuration

  <ldap-authentication-provider user-dn-pattern="uid={0},ou=people"
          group-search-base="ou=groups" />
    
and authenticated successfully as user “ben”, the subsequent loading of authorities would perform a search under the directory entry ou=groups,dc=springframework,dc=org, looking for entries which contain the attribute uniqueMember with value uid=ben,ou=people,dc=springframework,dc=org. By default the authority names will have the prefix ROLE_ prepended. You can change this using the role-prefix attribute. If you don't want any prefix, use role-prefix="none". For more information on loading authorities, see the Javadoc for the DefaultLdapAuthoritiesPopulator class.

18.4 Implementation Classes
The namespace configuration options we've used above are simple to use and much more concise than using Spring beans explicitly. There are situations when you may need to know how to configure Spring Security LDAP directly in your application context. You may wish to customize the behaviour of some of the classes, for example. If you're happy using namespace configuration then you can skip this section and the next one.

The main LDAP provider class, LdapAuthenticationProvider, doesn't actually do much itself but delegates the work to two other beans, an LdapAuthenticator and an LdapAuthoritiesPopulator which are responsible for authenticating the user and retrieving the user's set of GrantedAuthoritys respectively.

18.4.1 LdapAuthenticator Implementations
The authenticator is also responsible for retrieving any required user attributes. This is because the permissions on the attributes may depend on the type of authentication being used. For example, if binding as the user, it may be necessary to read them with the user's own permissions.

There are currently two authentication strategies supplied with Spring Security:

Authentication directly to the LDAP server ("bind" authentication).

Password comparison, where the password supplied by the user is compared with the one stored in the repository. This can either be done by retrieving the value of the password attribute and checking it locally or by performing an LDAP "compare" operation, where the supplied password is passed to the server for comparison and the real password value is never retrieved.

Common Functionality
Before it is possible to authenticate a user (by either strategy), the distinguished name (DN) has to be obtained from the login name supplied to the application. This can be done either by simple pattern-matching (by setting the setUserDnPatterns array property) or by setting the userSearch property. For the DN pattern-matching approach, a standard Java pattern format is used, and the login name will be substituted for the parameter {0}. The pattern should be relative to the DN that the configured SpringSecurityContextSource will bind to (see the section on connecting to the LDAP server for more information on this). For example, if you are using an LDAP server with the URL ldap://monkeymachine.co.uk/dc=springframework,dc=org, and have a pattern uid={0},ou=greatapes, then a login name of "gorilla" will map to a DN uid=gorilla,ou=greatapes,dc=springframework,dc=org. Each configured DN pattern will be tried in turn until a match is found. For information on using a search, see the section on search objects below. A combination of the two approaches can also be used - the patterns will be checked first and if no matching DN is found, the search will be used.

BindAuthenticator
The class BindAuthenticator in the package org.springframework.security.ldap.authentication implements the bind authentication strategy. It simply attempts to bind as the user.

PasswordComparisonAuthenticator
The class PasswordComparisonAuthenticator implements the password comparison authentication strategy.

Active Directory Authentication
In addition to standard LDAP authentication (binding with a DN), Active Directory has its own non-standard syntax for user authentication.

18.4.2 Connecting to the LDAP Server
The beans discussed above have to be able to connect to the server. They both have to be supplied with a SpringSecurityContextSource which is an extension of Spring LDAP's ContextSource. Unless you have special requirements, you will usually configure a DefaultSpringSecurityContextSource bean, which can be configured with the URL of your LDAP server and optionally with the username and password of a "manager" user which will be used by default when binding to the server (instead of binding anonymously). For more information read the Javadoc for this class and for Spring LDAP's AbstractContextSource.

18.4.3 LDAP Search Objects
Often a more complicated strategy than simple DN-matching is required to locate a user entry in the directory. This can be encapsulated in an LdapUserSearch instance which can be supplied to the authenticator implementations, for example, to allow them to locate a user. The supplied implementation is FilterBasedLdapUserSearch.

FilterBasedLdapUserSearch
This bean uses an LDAP filter to match the user object in the directory. The process is explained in the Javadoc for the corresponding search method on the JDK DirContext class. As explained there, the search filter can be supplied with parameters. For this class, the only valid parameter is {0} which will be replaced with the user's login name.

18.4.4 LdapAuthoritiesPopulator
After authenticating the user successfully, the LdapAuthenticationProvider will attempt to load a set of authorities for the user by calling the configured LdapAuthoritiesPopulator bean. The DefaultLdapAuthoritiesPopulator is an implementation which will load the authorities by searching the directory for groups of which the user is a member (typically these will be groupOfNames or groupOfUniqueNames entries in the directory). Consult the Javadoc for this class for more details on how it works.

If you want to use LDAP only for authentication, but load the authorities from a difference source (such as a database) then you can provide your own implementation of this interface and inject that instead.

18.4.5 Spring Bean Configuration
A typical configuration, using some of the beans we've discussed here, might look like this:

<bean id="contextSource"
        class="org.springframework.security.ldap.DefaultSpringSecurityContextSource">
  <constructor-arg value="ldap://monkeymachine:389/dc=springframework,dc=org"/>
  <property name="userDn" value="cn=manager,dc=springframework,dc=org"/>
  <property name="password" value="password"/>
</bean>

<bean id="ldapAuthProvider"
    class="org.springframework.security.ldap.authentication.LdapAuthenticationProvider">
 <constructor-arg>
   <bean class="org.springframework.security.ldap.authentication.BindAuthenticator">
     <constructor-arg ref="contextSource"/>
     <property name="userDnPatterns">
       <list><value>uid={0},ou=people</value></list>
     </property>
   </bean>
 </constructor-arg>
 <constructor-arg>
   <bean
     class="org.springframework.security.ldap.userdetails.DefaultLdapAuthoritiesPopulator">
     <constructor-arg ref="contextSource"/>
     <constructor-arg value="ou=groups"/>
     <property name="groupRoleAttribute" value="ou"/>
   </bean>
 </constructor-arg>
</bean>
                
This would set up the provider to access an LDAP server with URL ldap://monkeymachine:389/dc=springframework,dc=org. Authentication will be performed by attempting to bind with the DN uid=<user-login-name>,ou=people,dc=springframework,dc=org. After successful authentication, roles will be assigned to the user by searching under the DN ou=groups,dc=springframework,dc=org with the default filter (member=<user's-DN>). The role name will be taken from the “ou” attribute of each match.

To configure a user search object, which uses the filter (uid=<user-login-name>) for use instead of the DN-pattern (or in addition to it), you would configure the following bean

<bean id="userSearch"
    class="org.springframework.security.ldap.search.FilterBasedLdapUserSearch">
  <constructor-arg index="0" value=""/>
  <constructor-arg index="1" value="(uid={0})"/>
  <constructor-arg index="2" ref="contextSource" />
</bean> 
                
and use it by setting the BindAuthenticator bean's userSearch property. The authenticator would then call the search object to obtain the correct user's DN before attempting to bind as this user.

18.4.6 LDAP Attributes and Customized UserDetails
The net result of an authentication using LdapAuthenticationProvider is the same as a normal Spring Security authentication using the standard UserDetailsService interface. A UserDetails object is created and stored in the returned Authentication object. As with using a UserDetailsService, a common requirement is to be able to customize this implementation and add extra properties. When using LDAP, these will normally be attributes from the user entry. The creation of the UserDetails object is controlled by the provider's UserDetailsContextMapper strategy, which is responsible for mapping user objects to and from LDAP context data:

public interface UserDetailsContextMapper {
  UserDetails mapUserFromContext(DirContextOperations ctx, String username,
          Collection<GrantedAuthority> authorities);

  void mapUserToContext(UserDetails user, DirContextAdapter ctx);
}
                
Only the first method is relevant for authentication. If you provide an implementation of this interface and inject it into the LdapAuthenticationProvider, you have control over exactly how the UserDetails object is created. The first parameter is an instance of Spring LDAP's DirContextOperations which gives you access to the LDAP attributes which were loaded during authentication. The username parameter is the name used to authenticate and the final parameter is the collection of authorities loaded for the user by the configured LdapAuthoritiesPopulator.

The way the context data is loaded varies slightly depending on the type of authentication you are using. With the BindAuthenticator, the context returned from the bind operation will be used to read the attributes, otherwise the data will be read using the standard context obtained from the configured ContextSource (when a search is configured to locate the user, this will be the data returned by the search object).

Prev 	Up	 Next
17. Pre-Authentication Scenarios 	Home	 19. JSP Tag Libraries















#Introduction

This Docker image provides an [ApacheDS](https://directory.apache.org/apacheds/) LDAP server. Optionally it could be used to provide a [Kerberos server](https://directory.apache.org/apacheds/advanced-ug/2.1-config-description.html#kerberos-server) as well.

The project sources can be found on [GitHub](https://github.com/g17/ApacheDS). The Docker image on [Docker Hub](https://registry.hub.docker.com/u/h3nrik/apacheds/).


#Build

	git clone https://github.com/g17/ApacheDS.git
    docker build -t h3nrik/apacheds .


#Installation

The folder */var/lib/apacheds-${APACHEDS_VERSION}* contains the runtime data and thus has been defined as a volume. A [volume container](https://docs.docker.com/userguide/dockervolumes/) could be used for that. The image uses exactly the file system structure defined by the [ApacheDS documentation](https://directory.apache.org/apacheds/advanced-ug/2.2.1-debian-instance-layout.html).

The container can be started issuing the following command:

    docker run --name ldap -d -p 389:10389 h3nrik/apacheds


#Usage

You can manage the ldap server with the admin user *uid=admin,ou=system* and the default password *secret*. The *default* instance comes with a pre-configured partition *dc=example,dc=com*.

An indivitual admin password should be set following [this manual](https://directory.apache.org/apacheds/basic-ug/1.4.2-changing-admin-password.html).

Then you can import entries into that partition via your own *ldif* file. A [sample.ldif](https://github.com/g17/ApacheDS/blob/master/sample/sample.ldif) file is provided with the sources:

    ldapadd -v -h <your-docker-ip>:389 -c -x -D uid=admin,ou=system -w <your-admin-password> -f `pwd`/sample/sample.ldif


#Customization

It is also possible to start up your own defined Apache DS *instance* with your own configuration for *partitions* and *services*. Therefore you need to mount your [config.ldif](https://github.com/g17/ApacheDS/blob/master/instance/config.ldif) file and set the *APACHEDS_INSTANCE* environment variable properly. In the provided sample configuration the instance is named *default*. Assuming your custom instance is called *yourinstance* the following command will do the trick:

    docker run --name ldap -d -p 389:10389 -e APACHEDS_INSTANCE=yourinstance -v /path/to/your/config.ldif:/bootstrap/conf/config.ldif:ro h3nrik/apacheds


It would be possible to use this ApacheDS image to provide a [Kerberos server](https://directory.apache.org/apacheds/advanced-ug/2.1-config-description.html#kerberos-server) as well. Just provide your own *config.ldif* file for that. Don't forget to expose the right port, then.

Also other services are possible. For further information read the [configuration documentation](https://directory.apache.org/apacheds/advanced-ug/2.1-config-description.html).

