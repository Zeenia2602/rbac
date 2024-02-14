
**Following changes need to make in files to RBAC in broker, schema registry, and kafka-connect properties file**

  1. /etc/kafka/server.properties
     
             authorizer.class.name=io.confluent.kafka.security.authorizer.ConfluentServerAuthorizer
              super.users=User:localhost;User:ANONYMOUS
              ssl.principal.mapping.rules=RULE:^CN=(.*),OU=.*$/$1/,DEFAULT
              confluent.license.topic.replication.factor=1
              provider for RBAC and centralized ACLs
              confluent.authorizer.access.rule.providers=ZK_ACL,CONFLUENT
              confluent.metadata.topic.replication.factor=1
              confluent.security.event.logger.exporter.kafka.topic.replicas=1
              confluent.metadata.server.listeners=http://0.0.0.0:8090
              confluent.metadata.server.advertised.listeners=http://localhost:8090
              confluent.metadata.server.authentication.method=BEARER
              confluent.metadata.server.token.key.path=/home/platformatory/Documents/rbac/rbac-token.pem
              confluent.metadata.server.user.store=FILE
              confluent.metadata.server.user.store.file.path=/home/platformatory/Documents/rbac/login.properties
              listener.name.token.oauthbearer.sasl.login.callback.handler.class=io.confluent.kafka.server.plugins.auth.token.TokenBearerServerLoginCallbackHandler
              listener.name.token.oauthbearer.sasl.server.callback.handler.class=io.confluent.kafka.server.plugins.auth.token.TokenBearerValidatorCallbackHandler
              listener.name.token.oauthbearer.sasl.jaas.config= \
                 org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
                 publicKeyPath="/home/platformatory/Documents/rbac/rbac-public.pem";
              listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL,TOKEN:SASL_PLAINTEXT
              listener.name.token.sasl.enabled.mechanisms=OAUTHBEARER
    
  2. /etc/schema-registry/schema-registry.properties
       
              kafkastore.security.protocol=SASL_PLAINTEXT
              kafkastore.sasl.mechanism=OAUTHBEARER
              
              kafkastore.sasl.login.callback.handler.class=io.confluent.kafka.clients.plugins.auth.token.TokenUserLoginCallbackHandler
              kafkastore.sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
                username="sr" \
                password="schema" \
                metadataServerUrls="http://localhost:8090";
              
              schema.registry.group.id=schema-registry
              ssl.keystore.location=/home/platformatory/Documents/rbac/sr/schema.keystore.jks
              ssl.keystore.password=schema
              ssl.truststore.location=/home/platformatory/Documents/rbac/sr/client.truststore.jks
              ssl.truststore.password=secret
              ssl.key.password=schema
              // Security plugin
              resource.extension.class=io.confluent.kafka.schemaregistry.security.SchemaRegistrySecurityResourceExtension
              confluent.schema.registry.authorizer.class=io.confluent.kafka.schemaregistry.security.authorizer.rbac.RbacAuthorizer
                        
              rest.servlet.initializor.classes=io.confluent.common.security.jetty.initializer.InstallBearerOrBasicSecurityHandler
              
              // location of running metadata service 
              confluent.metadata.bootstrap.server.urls=http://localhost:8090
              
              #credentials to use when communicate with MDS 
              confluent.metadata.basic.auth.user.info=sr:schema
              confluent.metadata.http.auth.credentials.provider=BASIC
              
              // Path of public keys 
              public.key.path=/home/platformatory/Documents/rbac/rbac-public.pem
              
              // Enable Anonymous access with a principal of User:ANOYMOUS
              confluent.schema.registry.anonymous.principal=true
              authentication.skip.paths=/*
     
  4. /etc/kafka/connect-distributed.properties
    
              security.protocol=SASL_PLAINTEXT
              sasl.mechanism=OAUTHBEARER
              sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
                username="connect" \
                password="connect" \
                metadataServerUrls="http://localhost:8090";
              sasl.login.callback.handler.class=io.confluent.kafka.clients.plugins.auth.token.TokenUserLoginCallbackHandler
              ssl.keystore.location=/home/platformatory/Documents/rbac/connectors/connect.keystore.jks
              ssl.keystore.password=connect
              ssl.truststore.location=/home/platformatory/Documents/rbac/connectors/client.truststore.jks
              ssl.truststore.password=secret
              ssl.key.password=connect
              
              rest.servlet.initializor.classes=io.confluent.common.security.jetty.initializer.InstallBearerOrBasicSecurityHandler
              public.key.path=/home/platformatory/Documents/rbac/rbac-public.pem
              confluent.metadata.bootstrap.server.urls=http://localhost:8090
              confluent.metadata.basic.auth.user.info=connect:connect
              confluent.metadata.http.auth.credentials.provider=BASIC

# Create the login.properties file

      localhost:secret
      consumer:consumer
      connect:connect
      sr:schema
      
# Create the kafka.properties file for OAUTH Bearer authentication

      sasl.mechanism=OAUTHBEARER
      security.protocol=SASL_PLAINTEXT
      sasl.login.callback.handler.class=io.confluent.kafka.clients.plugins.auth.token.TokenUserLoginCallbackHandler
      sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required username="consumer" password="consumer" metadataServerUrls="http://localhost:8090";
           
# Login into the confluent cli using the following command 

      confluent login --url http://localhost:8090

# To the kafka-cluster id, Use this command: 

      confluent cluster describe --url http://localhost:8090

# Assign the RBAC role binding to the user

  NOTE: 
    
    Make sure this users should be in your login.properties file 
    
  1.  **RBAC for kafka broker**

       NOTE:

         Make sure the your kafka-server and kafka-zookeeper is running before creating any role-bindings.

      a. Grant the SystemAdmin role to your User, which you want to have the full access just like your superuser

            confluent iam rbac role-binding create --principal User:localhost --role SystemAdmin --kafka-cluster <kafka-cluster-id>

        **Example:**

            confluent iam rbac role-binding create --principal User:localhost --role SystemAdmin --kafka-cluster IfsjpqubT2aWb645x4iimw

       b. List the role - binding for that user
     
            confluent iam rbac role-binding list --principal User:localhost --kafka-cluster IfsjpqubT2aWb645x4iimw
     
       c. Grant the ResourceOwner role to your client user, so that it can create a topic

            confluent iam rbac role-binding create --principal User:consumer --role ResourceOwner --resource Topic:rbac1 --kafka-cluster IfsjpqubT2aWb645x4iimw
                 
       d. Grant the DeveloperRead and DeveloperWrite to you client user with another topic who is not an ResourceOwner.

            confluent iam rbac role-binding create --principal User:consumer --role DeveloperWrite --resource Topic:rbac-test --kafka-cluster IfsjpqubT2aWb645x4iimw
            confluent iam rbac role-binding create --principal User:consumer --role DeveloperRead --resource Topic:rbac --kafka-cluster IfsjpqubT2aWb645x4iimw
            confluent iam rbac role-binding create --principal User:consumer --role DeveloperRead --resource Group:console-consumer --prefix --kafka-cluster IfsjpqubT2aWb645x4iimw
     
  2. **RBAC for Schema Registry**

     Note:
       After providing all the RBAC role-bindings, you need to start the schema registry clusters.

      a. Grant the ResourceOwner role to these topics: _schemas, _schema_encoders, _dek_registry_keys

             confluent iam rbac role-binding create --principal User:<schema-registry-user> --role ResourceOwner --resource Topic:_schemas --kafka-cluster <kafka-cluster-id>

     **Example**

             confluent iam rbac role-binding create --principal User:sr --role ResourceOwner --resource Topic:_schemas --kafka-cluster IfsjpqubT2aWb645x4iimw
             confluent iam rbac role-binding create --principal User:sr --role ResourceOwner --resource Topic:_schema_encoders --kafka-cluster IfsjpqubT2aWb645x4iimw
             confluent iam rbac role-binding create --principal User:sr --role ResourceOwner --resource Topic:_dek_registry_keys --kafka-cluster IfsjpqubT2aWb645x4iimw
             
      b. Grant the ResourceOwner role to the schema-registry group

             confluent iam rbac role-binding create --principal User:sr --role ResourceOwner --resource Group: schema-registry --kafka-cluster IfsjpqubT2aWb645x4iimw
     
      c. Grant the DeveloperRead and DeveloperWrite role to Topic: _confluent-command

             confluent iam rbac role-binding create --principal User:sr --role DeveloperWrite --resource Topic:_confluent-command --kafka-cluster IfsjpqubT2aWb645x4iimw
             confluent iam rbac role-binding create --principal User:sr --role DeveloperRead --resource Topic:_confluent-command --kafka-cluster IfsjpqubT2aWb645x4iimw
     
      d. List the role bindings for schema registry User.

             confluent iam rbac role-binding list --principal User:sr --kafka-cluster IfsjpqubT2aWb645x4iimw
     
  4. **RBAC for kafka-connect**

       a. Grant the ResourceOwner role to the these topics: connect-configs, connect-offsets, and connect-status.

            confluent iam rbac role-binding create --principal User:<connect-user> --role ResourceOwner --resource Topic:connect-configs --kafka-cluster <kafka-cluster-id>
            confluent iam rbac role-binding create --principal User:<connect-user> --role ResourceOwner --resource Topic:connect-status --kafka-cluster <kafka-cluster-id>
            confluent iam rbac role-binding create --principal User:<connect-user> --role ResourceOwner --resource Topic:connect-offsets --kafka-cluster <kafka-cluster-id>

     **Example:**

            confluent iam rbac role-binding create --principal User:connect --role ResourceOwner --resource Topic:connect-configs --kafka-cluster IfsjpqubT2aWb645x4iimw
            confluent iam rbac role-binding create --principal User:connect --role ResourceOwner --resource Topic:connect-offsets --kafka-cluster IfsjpqubT2aWb645x4iimw
            confluent iam rbac role-binding create --principal User:connect --role ResourceOwner --resource Topic:connect-status --kafka-cluster IfsjpqubT2aWb645x4iimw
             
       b. Grant the ResourceOwner tole to the the connect cluster group.

            confluent iam rbac role-binding create --principal User:connect --role ResourceOwner --resource Group:connect-cluster --kafka-cluster IfsjpqubT2aWb645x4iimw
     
       c. Install the kafka connector

            confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:<datagen-version>
            
       d. Grant the SecurityAdmin role to Connect cluster

            confluent iam rbac role-binding create --principal User:connect --role SecurityAdmin  --kafka-cluster IfsjpqubT2aWb645x4iimw --connect-cluster connect-cluster

          Note:
            Start the connect cluster, before granting the Security Admin role to connect cluster

           
       e. Grant the ResourceOwner role to Connector

             confluent iam rbac role-binding create --principal User:connect --role ResourceOwner --resource Connector:datagen-pageviews --kafka-cluster IfsjpqubT2aWb645x4iimw --connect-cluster connect-cluster
     
       f. Grant the ResourceOwner role to topic

             confluent iam rbac role-binding create --principal User:connect --role
     
       g. Grant the DeveloperRead to Consumer Group

             confluent iam rbac role-binding create --principal User:connect --role DeveloperRead --resource Group:console-consumer- --prefix --kafka-cluster IfsjpqubT2aWb645x4iimw
     
            
