# IntelliCert

A small program which creates a JKS(Java keystore) that can be directly used in Java code to connect to a ssl endpoint.

Steps to be followed:
 * Download this repo on your local.
 * Compile this code.
 * Run this code while providing domain name and port.
 * It generates a keystore file with all required certificates.
 * Import keystore in your project and create and pass SSL Context.
  
 This saves a lot of efforts when connecting to remove systems such as webserver, api gateways, databases (mysql, mongodb, etc) over tls. Especially solves issues when connecting to cloud based solutions - AWS, Azure, etc.

## Introduction

Java program written by Andreas Sterbenz, and posted on a blog in Oct, 2006:
https://blogs.oracle.com/gc/entry/unable_to_find_valid_certification

Link to Java program in Andreas' blog no longer works, but the source was linked in another blog:
https://web.archive.org/web/20190831085142/http://nodsw.com/blog/leeland/2006/12/06-no-more-unable-find-valid-certification-path-requested-target

Usage:
Need to compile, first:
javac InstallCert.java

Note: since java 11, you can run it directly without compiling it first:
java --source 11 InstallCert.java <args>

### Access server, and retrieve certificate (accept default certificate 1)
java InstallCert [--proxy=proxyHost:proxyPort] <host>[:port] [passphrase]

### Extract certificate from created jssecacerts keystore
keytool -exportcert -alias [host]-1 -keystore jssecacerts -storepass changeit -file [host].cer

### Import certificate into system keystore
keytool -importcert -alias [host] -keystore [path to system keystore] -storepass changeit -file [host].cer

## Example:
java InstallCert woot.com:443

    Loading KeyStore /usr/lib/jvm/java-6-sun-1.6.0.26/jre/lib/security/cacerts...
    Opening connection to woot.com:443...
    Starting SSL handshake...

    javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

    <...>

    Server sent 1 certificate(s):

     1 Subject O=Woot Inc, C=US, ST=Texas, L=Carrollton, CN=*.woot.com
       Issuer  CN=SecureTrust CA, O=SecureTrust Corporation, C=US
       sha1    4b 46 ca 6b 83 05 b3 51 ff c6 e7 9c fd b3 9b e3 3f 2e c4 53 
       md5     e8 a5 88 1b d5 67 bb fc 88 cc b1 c5 2b ac c4 7d 

    Enter certificate to add to trusted keystore or 'q' to quit: [1]

[enter]

    [
    [
      Version: V3
      Subject: O=Woot Inc, C=US, ST=Texas, L=Carrollton, CN=*.woot.com
      Signature Algorithm: SHA1withRSA, OID = 1.2.840.113549.1.1.5

    <...>

    Added certificate to keystore 'jssecacerts' using alias 'woot.com-1'

keytool -exportcert -alias woot.com-1 -keystore jssecacerts -storepass changeit -file woot.com.cer

    Certificate stored in file <woot.com.cer>
  
(sudo) keytool -importcert -alias woot.com -keystore /usr/lib/jvm/java-6-sun-1.6.0.26/jre/lib/security/cacerts -storepass changeit -file woot.com.cer

    Owner: O=Woot Inc, C=US, ST=Texas, L=Carrollton, CN=*.woot.com
    Issuer: CN=SecureTrust CA, O=SecureTrust Corporation, C=US
  
    <...>
  
    Trust this certificate? [no]:
  
yes

    Certificate was added to keystore

## Using generated keystore in Spring boot project

Copy jssecacerts.jks in resource folder of Spring boot project.
Create following MongoClient in Configuration extending AbstractMongoClientConfiguration.
    
    @Value("${mongo.config.connectionString}")
    private String connectionString;
    
    @Value("${mongo.config.keystore.password}")
    private String keyStorePassword;

    public MongoClient mongoClient() {

    File file = ResourceUtils.getFile("classpath:jssecacerts.jks");
    InputStream inputStream = new FileInputStream(file);

    KeyStore keyStore = KeyStore.getInstance("JKS");
    keyStore.load(inputStream, keyStorePassword.toCharArray());

    TrustManagerFactory trustManagerFactory = TrustManagerFactory
        .getInstance(TrustManagerFactory.getDefaultAlgorithm());
    trustManagerFactory.init(keyStore);

    SSLContext sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, trustManagerFactory.getTrustManagers(), null);

    MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
        .applyToSslSettings(builder -> {builder.enabled(true).invalidHostNameAllowed(true).context(sslContext);})
        .applyConnectionString(new ConnectionString(connectionString))
        .build();

    return MongoClients.create(mongoClientSettings);
        
    }

## Other relevant tools

Keystore Explorer for creating Java Keystore manually:

https://braytons.medium.com/importing-aws-rds-pem-certificate-to-java-keystore-using-keystore-explorer-606a5a5236fa
http://keystore-explorer.org/downloads.html
    
