---
layout: master
tile: Blog
full_posts: 3
---

Accessing Custom Liberty Feature Service Interfaces via Injection
===================

For a few releases now IBM Webpshere Liberty Profile has been available as light weight platform for web applications.  Included in it is support for Enterprise OSGi 4.2 Blueprint applications and OSGi web applications.  The advantages of these over traditional Java EE web application development models are modularity and service dynamism which allows applications to be composed of as an assembly of modular components with dependencies resolved at runtime.  There are many well written articles online which elaborate on these topics, so I will not go into them here.

After embracing Enterprise OSGi applications in Liberty, often you reach a point where you want one EBA application to provide services to the other EBA applications.  A common use case would be a configuration service or some other kind of resource broker.  Unfortunately in these cases, each EBA application is isolated from the others and its services are not visible to them.  Starting with Liberty 8.5.x you can overcome this issue using its Service Provider Interface.  This interface is well documented in the IBM [infocenter] (http://www-01.ibm.com/support/knowledgecenter/SSEQTP_8.5.5/com.ibm.websphere.wlp.core.doc/ae/twlp_feat_develop.html).

While I have been aware of this capability for some time, it was not until discussing a configuration broker service with IBM's Graham Charters that I realized that SPI features could have their service interfaces injected into EBA applications using Blueprint.  In the course of my day job, this pattern has emerged a few times so I thought it would be good to share the know how and concept broadly.

I developed the example service and consumer detailed below using Eclipse 4.4 Luna with the [Websphere Development Tools](https://developer.ibm.com/assets/wasdev/#filter/sortby=relevance;q=Websphere%20Developer%20Tools) plugin(s) installed.  It is possible to do it without the plugins, but they do keep out of a bit of trouble by generating some of the manifests for you.  Included in the example is an Ant build file which packages up the artifacts for deployment without relying on any plugins.

The SPI feature consists of three Java files and associated bundle and feature manifests.  One java file, HelloWorld.java defines the Service Interface while another, HelloWorldImpl.java implements the interface.  The third Java file, Activator.java, is the interesting one; it is the OSGi Bundle Activator which will be called by Liberty when it is time to start and stop the service.  On startup, it creates an instance of the service and registers it into the service registry in Liberty.  The startup method is shown below.

```java
	/*
	 * (non-Javadoc)
	 * @see org.osgi.framework.BundleActivator#start(org.osgi.framework.BundleContext)
	 */
	public void start(BundleContext bundleContext) throws Exception {
		Activator.context = bundleContext;
		Hashtable<String,String> metadata = new Hashtable<String,String>();
		metadata.put("service.description","Example hello Liberty SPR Blueprint Service");
		metadata.put("service.vendor", "example.org");
		String[] interfaces = {HelloWorld.class.getName()};
		registration = context.registerService(interfaces, new HelloWorldImpl(), metadata);
		System.out.println("HelloWorld service registered to OSGi");
	}
```

When called, this method creates an instance of the service, associates it with the service interface, and registers it into the OSGi Service Registry.

Now lets look at the Liberty feature manifest, **SUBSYSTEM.MF**.  The contents of this file are documented on the Infocenter and this file can be generated for you by the WebSphere Development Tools.

```
    Subsystem-ManifestVersion: 1.0
    IBM-Feature-Version: 2
    IBM-ShortName: exampleFeature-1.0
    Subsystem-SymbolicName: org.example.liberty.feature;visibility:=public
    Subsystem-Version: 1.0.0
    Subsystem-Type: osgi.subsystem.feature
    Manifest-Version: 1.0
    Subsystem-Content: org.example.liberty.service;version=1.0.0
    IBM-API-Package: org.example.liberty.service;type="api"
    IBM-API-Service: org.example.liberty.service.HelloWorld
```

The notable lines here are the last three.  **Subsystem-Content** references the name of the OSGi Bundle containing the feature code.  **IBM-API-Package** is a comma separated list of Java packages which should be made available to the ClassLoaders of deployed applications.  And finally, **IBM-API-Service** is a comma separated list of Java Interfaces which should be exposed as OSGi services to deployed applications.

For the consumer, the most interesting thing is the blueprint.xml.  When the consumer application is deployed, Liberty processes the blueprint.xml file and creates the consumer components and injects the references to the service interface published by the SPI feature.  Here is the blueprint.xml for this example application.

```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0" >
      <reference id="helloService" interface="org.example.liberty.service.HelloWorld"/>
      <bean id="helloClientBean" class="org.example.liberty.client.HelloClient" 
         activation="eager" init-method="init" destroy-method="destroy" >
  	    <property name="helloWorld" ref="helloService" />
      </bean>
    </blueprint>
```

When processed, the **`<bean>`** stanza causes Blueprint to create an instance of the consumer while the enclosed **`<property>`** element asks Blueprint to inject an instance of the helloService.  In the **`<reference>`** element the helloService is defined to have a service interface of org.example.liberty.service.HelloWorld which matches the interface we exposed in **IBM-API-Service** in the SUBSYSTEM.MF above.

To deploy these artifacts, an ESA file and an EBA file are created.  The EBA file contains the consumer application and the ESA file contains the feature or subsystem.  The application can be deployed either as a dropin or using an **`<application>`** directive in Liberty's server.xml.  The ESA file is deployed using the **featureManager** command included in the Liberty **bin** directory.

I have exported the Eclipse project containing everything required to build and run the example.  It is available at this [link](/downloads/org.example.zip).  I hope that you find this technique useful in your Liberty Profile based solutions.

