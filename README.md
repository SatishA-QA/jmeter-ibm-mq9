# jmeter-ibm-mq9

Simplest examples of JMeter 5.3 with Dockerized IBM MQ 9.1.2.0.

## Setup MQ Service Script

A bash script to start the default MQ.   This is not persistent but it can be configured to be persistent.
This command might need a tweak of the --volume option in order to work on Windows.

```
#!/bin/bash
## version is 9.1.2.0 at time of this documentation
docker run \
  --name mqdemo \
  --env LICENSE=accept \
  --env MQ_QMGR_NAME=QM1 \
  --env MQ_ENABLE_METRICS=true \
  --env MQ_ENABLE_EMBEDDED_WEB_SERVER=true \
  --publish 1414:1414 \
  --publish 9009:9443 \
  --publish 9157:9157 \
  --volume "$PWD/cert/mykey:/etc/mqm/pki/keys/mykey" \
  --detach \
  ibmcom/mq
```

#### Other options you can include

- MQ_DEV - Set this to false to stop the default objects being created.
- MQ_ADMIN_PASSWORD - Changes the password of the admin user. Must be at least 8 characters long.
- MQ_APP_PASSWORD

#### Related Info

* https://github.com/ibm-messaging/mq-container/blob/master/docs/usage.md
* https://github.com/ibm-messaging/mq-container/blob/master/docs/developer-config.md

#### Administration

- Admin console so you can watch the queue:  https://localhost:9009/ibmmq/console
    -  User: admin
    -  Password: passw0rd

* Prometheus metrics:  http://localhost:9157/metrics

#### Setup Certs And SSL

TODO:  Need to create an `mq_client.ks`  that matches the cert served by the MQ server.

https://developer.ibm.com/components/ibm-mq/tutorials/mq-secure-msgs-tls/

##### What I did

Use the Java `Keytool Explorer` app.   Here are the steps I used:

* Create a new `Server keypair` as `mq_client.jks` ,  2048 bit RSA and 10 years long.
* Export the `certificate chain` as `tls.crt` in DER format.
* Export the PKCS#8 `private key` as `tls.key`
* Load those two files into the mykey folder shown in the Docker config above.


##### If you setup MQ to be SSL

Start JMeter like so, using the included `sslExample.jmx` project file:

    ./bin/jmeter -J"jmsPassword=passw0rd" -J"jksPassword=changeme"
    
When you start JMeter, the `mq_client.jks` must be in the same folder as the project file.


## JMeter Setup

This example uses JMeter 5.3 .  You can download from the usual location.

The following 3 libs need to be loaded in JMeter lib/ext folder:

- the JMeter plugin manager
- com.ibm.mq.allclient-9.1.5.0.jar
- javax.jms-api-2.0.1.jar

After that, make sure you use the plugins manger to add all the plugins that contain extra charts and graphs.

Then, you will be able to load the project file.

* https://www.blazemeter.com/blog/ibm-mq-testing-with-jmeter-learn-how

## Included Project Files

- simplestExample.jmx - will only work when MQ is configured with no password and no SSL
     - in this case, when you start the MQ server, don't include the volume mapping
- sslExample.jmx - after enabling SSL in the MQ server, this project includes code to connect with a keystore


## What It Looks Like

![JMeter Test Plan Example](https://github.com/djangofan/jmeter-ibm-mq9/blob/master/jmeterGui.png)



# Prototype Code

Here is the prototype code when I initially got this working.  The code currently checked into this repo may have been updated since then.

## SSL Groovy Client Code

Use an if-condition to ensure this only executes once during a multi-threaded test.

```
import com.ibm.msg.client.jms.JmsConnectionFactory
import com.ibm.msg.client.jms.JmsFactoryFactory
import com.ibm.msg.client.wmq.WMQConstants
import javax.net.ssl.SSLSocketFactory
import javax.net.ssl.SSLContext
import javax.net.ssl.KeyManagerFactory
import javax.net.ssl.TrustManagerFactory
import javax.net.ssl.CertPathTrustManagerParameters
import javax.jms.Session
import java.security.KeyStore
import java.security.SecureRandom
import java.security.cert.CertPathBuilder
import java.security.cert.PKIXRevocationChecker
import java.security.cert.PKIXBuilderParameters
import java.security.cert.X509CertSelector

log.info("#### Start... ")
log.info("JMeter Project File Location: " + vars.get("projectHome"))
log.info("JMeter Home: " + vars.get("jmeterHome"))
//System.setProperty("javax.net.debug", "ssl:handshake")
//System.setProperty("javax.net.debug", "ssl:record")

SSLContext sslContext() {
    def jksPassword = vars.get("jksPass").toCharArray()
    log.info("Password: " + jksPassword)
    File file = new File(vars.get("projectHome") + "/" + vars.get("jksFile"))
    def jksPath = file.getAbsolutePath()
    log.info("JKS Loaded: " + jksPath)

    new FileInputStream(jksPath).with { cert ->
       try {
       KeyStore caCertsKeyStore = KeyStore.getInstance("JKS")
        caCertsKeyStore.load(cert, jksPassword)

        KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm())
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm())

        CertPathBuilder cpb = CertPathBuilder.getInstance("PKIX")
        PKIXRevocationChecker rc = (PKIXRevocationChecker) cpb.getRevocationChecker()
        rc.setOptions(EnumSet.of(
            PKIXRevocationChecker.Option.PREFER_CRLS,
            PKIXRevocationChecker.Option.ONLY_END_ENTITY,
            PKIXRevocationChecker.Option.SOFT_FAIL,
            PKIXRevocationChecker.Option.NO_FALLBACK))
                    
        PKIXBuilderParameters pkixParams = new PKIXBuilderParameters(caCertsKeyStore, new X509CertSelector())
        pkixParams.addCertPathChecker(rc)

        kmf.init(caCertsKeyStore, jksPassword)
        tmf.init(new CertPathTrustManagerParameters(pkixParams))

        SSLContext sslContext = SSLContext.getInstance("TLS")
        sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), new SecureRandom())

        log.info("Finished acquiring ssl context.")
        return sslContext
       } catch(Exception e) {
       	log.error("Error loading SSL context.", e)
       } finally {
       	cert.close()
       }
    }
}

def ff = JmsFactoryFactory.getInstance(WMQConstants.WMQ_PROVIDER)
def cf = ff.createConnectionFactory()

cf.setHostName(vars.get("hostName"))
cf.setQueueManager(vars.get("queueManager"))
cf.setPort(Integer.parseInt(vars.get("hostPort")))
cf.setChannel(vars.get("channelName"))
cf.setTransportType(WMQConstants.WMQ_CM_CLIENT)
cf.setCCSID(1208)
cf.setAppName("jmeter")

cf.setBooleanProperty(WMQConstants.USER_AUTHENTICATION_MQCSP, true)
cf.setStringProperty(WMQConstants.WMQ_SSL_CIPHER_SPEC, vars.get("cipherSpec"))
cf.setStringProperty(WMQConstants.USERID, vars.get("jmsUser"))
cf.setStringProperty(WMQConstants.PASSWORD, vars.get("jmsPass"))
System.setProperty("com.ibm.mq.cfg.useIBMCipherMappings", "false")

SSLSocketFactory sslSocketFactory = sslContext().getSocketFactory()
cf.setSSLSocketFactory(sslSocketFactory)

def conn = cf.createConnection()
def sess = conn.createSession(false, Session.AUTO_ACKNOWLEDGE)

def destination = sess.createQueue(vars.get("queueName"))

conn.start()  // create producer or consumer instance after singleton connection is started

log.info("#### Connection created ...")

System.getProperties().put("Session", sess)
System.getProperties().put("Connection", conn)
System.getProperties().put("Destination", destination)

vars.put("setupDone", "true")
```

## Queue Producer Code

```
import java.time.Instant

def sess = System.getProperties().get("Session")
def destination = System.getProperties().get("Destination")

def producer = sess.createProducer(destination)
def rnd = new Random(System.currentTimeMillis())
def payload = String.format("JMeter...IBM MQ...test message no. %09d!", rnd.nextInt(Integer.MAX_VALUE))
def msg = sess.createTextMessage(payload)

def start = Instant.now()

producer.send(msg)
def stop = Instant.now()
producer.close()

SampleResult.setResponseData(msg.toString())
SampleResult.setDataType( org.apache.jmeter.samplers.SampleResult.TEXT )
SampleResult.setLatency( stop.toEpochMilli() - start.toEpochMilli() )
```

## Queue Consumer Code

```
import javax.jms.TextMessage
import javax.jms.BytesMessage

import java.time.LocalDate
import java.time.LocalTime
import java.time.Instant
import java.time.format.DateTimeFormatter

log.info("#### Looking for messages to consume...")
def sess = System.getProperties().get("Session")
def destination = System.getProperties().get("Destination")

def consumer = sess.createConsumer(destination)
def start = Instant.now()

def msg = consumer.receive(1000)
def stop = Instant.now()

if (msg != null) {
	if (msg instanceof BytesMessage) {
		def tmp = msg.asType(BytesMessage)
		log.debug("#### Incoming BytesMessage contains " + tmp.getBodyLength() + " bytes")
	} else if (msg instanceof TextMessage) {
		def tmp = msg.asType(TextMessage)
		log.debug("#### Incoming TextMessage contains -> " + tmp.getText())
	} else {
		log.debug("#### Incoming message has unexpected format!")
	}
	LocalDate date = LocalDate.parse(msg.getStringProperty("JMS_IBM_PutDate"),
					DateTimeFormatter.ofPattern("uuuuMMdd"))
	LocalTime time = LocalTime.parse(msg.getStringProperty("JMS_IBM_PutTime"),
					DateTimeFormatter.ofPattern("HHmmssSS"))

        def timestampDetail = String.format("#### Incoming message was placed in queue @ %s - %s", date, time)
	log.info(timestampDetail)

        SampleResult.setResponseData(msg.toString() + "\n\n" + timestampDetail)
	SampleResult.setDataType( org.apache.jmeter.samplers.SampleResult.TEXT )
	SampleResult.setLatency( stop.toEpochMilli() - start.toEpochMilli() )

} else {
	log.info("#### Nothing to fetch!")
}

consumer.close()

```

## Threads Stop Code

Use an if-condition to ensure this only executes once during a multi-threaded test.

```
System.getProperties().get("Session").close()
System.getProperties().get("Connection").close()
log.info("#### Stop completed!")
vars.put("stopDone", "true")
```
satish