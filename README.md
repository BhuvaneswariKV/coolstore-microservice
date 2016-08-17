Red Hat Cool Store Microservice Demo
====================================
This is an example demo showing a retail store consisting of a trio of microservices based on [JBoss EAP 7](https://access.redhat.com/products/red-hat-jboss-enterprise-application-platform/) and [Node.js](https://access.redhat.com/documentation/en/openshift-enterprise/3.2/paged/using-images/chapter-1-source-to-image-s2i), deployed to [OpenShift](https://access.redhat.com/products/openshift-enterprise-red-hat/) and protected with [Red Hat SSO](https://access.redhat.com/documentation/en/red-hat-single-sign-on/).

It demonstrates how to wire up small microservices into a larger application using microservice architectural principals.

Services
--------
There are three individual microservices that make up this app:

1. SSO Service - for protecting RESTful pricing service, using [Red Hat SSO](https://access.redhat.com/documentation/en/red-hat-single-sign-on/)
2. Pricing Service - Java EE application running on [JBoss EAP 7](https://access.redhat.com/products/red-hat-jboss-enterprise-application-platform/), serves prices for retail products
3. UI Service - A frontend based on [AngularJS](https://angularjs.org) and [PatternFly](http://patternfly.org) running in a [Node.js](https://access.redhat.com/documentation/en/openshift-enterprise/3.2/paged/using-images/chapter-1-source-to-image-s2i) container.

Demo Credentials and other identifiers
--------------------------------------
1. SSO Realm Name: `myrealm`
1. SSO / JBoss security role name: `user`
1. SSO Admin (used when logging into SSO Admin Console): username `admin` password: `admin`
1. Website User (used when accessing retail store): username: `appuser` password: `password`
1. SSO REST API Admin User (not generally used): username: `ssoservice` password: `ssoservicepass`

Running the Demo
================
Running the demo consists of 3 main steps, one for each of the services listed above. 

It is assumed you have installed OpenShift, either using [Red Hat's CDK](http://developers.redhat.com/products/cdk/overview/) or a complete install, and can login to the
web console or use the `oc` CLI tool.

Note: SSL/TLS Self-Signed Certificates
--------------------------------------
For demo purposes, you will most likely be using self-signed certificates, which will be apparent when
accessing the services using a browser (you'll get a security warning which must be accepted.)

Create project and associated service accounts and permissions
--------------------------------------------------------------

In the following steps, substitute your desired project name for PROJECT, and assume your OpenShift domain is DOMAIN.

1. Clone this repository

    git clone https://github.com/jamesfalkner/coolstore-microservice
    cd coolstore-microservice/openshift-templates
    
1. Login and Create a new project

    oc login https://DOMAIN:8443
    oc new-project PROJECT
    
1. Create OpenShift objects for SSL/TLS crypto secrets and associated service account:

    oc create -f secrets/coolstore-secrets.json

1. Add roles to service account to allow for kubernetes clustering access

    oc policy add-role-to-user view system:serviceaccount:PROJECT:default -n PROJECT
    oc policy add-role-to-user view system:serviceaccount:PROJECT:sso-service-account -n PROJECT

Deploy SSO service on OpenShift using the OpenShift `oc` CLI
------------------------------------------------------------

This will build and deploy an instance of Red Hat SSO server (based on [Keycloak](https://keycloak.org)) using
Red Hat's [xPaaS SSO image](https://access.redhat.com/documentation/en/red-hat-xpaas/version-0/red-hat-xpaas-sso-image/).

1. Create and deploy SSO service, wait for it to complete.

    oc process -f sso-service.json | oc create -f -

    You can view the process of the deployment using
    
    oc logs -f dc/sso
    
1. Once it completes, you can test it by accessing https://secure-sso-PROJECT.DOMAIN/auth or clicking on the associated route from the project overview page within the OpenShift web console.

1. Obtain the public key for the automatically-created realm `myrealm` by visiting https://secure-sso-PROJECT.DOMAIN/auth/realms/myrealm in your browser. You'll need in the next steps.

Deploy Pricing Service using the OpenShift `oc` CLI
---------------------------------------------------

1. Create and deploy service, substituting values for SSO_URL (don't forget the `/auth` suffix) and SSO_PUBLIC_KEY, wait for it to complete. 

    oc process -f pricing-service.json \
      SSO_URL=https://secure-sso-PROJECT.DOMAIN/auth \
      SSO_PUBLIC_KEY=<PUBLIC_KEY> | \
      oc create -f -

If you have created a [local Maven mirror](https://blog.openshift.com/improving-build-time-java-builds-openshift/) to speed up your builds, specify it with `MAVEN_MIRROR_URL` in the above command. 

1. Wait for it to complete (this step may take a while as it downloads all Maven dependencies during the build). Follow the logs using

    oc logs -f bc/pricing
 
Deploy the UI Service using the OpenShift `oc` CLI
--------------------------------------------------

1. Create and deploy service, substituting the appropriate values for the various services:

    oc process -f ui-service.json \
      SSO_URL=https://secure-sso-PROJECT.DOMAIN/auth \
      SSO_PUBLIC_KEY='<PUBLIC KEY>' \
      HOSTNAME_HTTP=ui-PROJECT.DOMAIN \
      HOSTNAME_HTTPS=secure-ui-PROJECT.DOMAIN \
      REST_ENDPOINT=http://pricing-PROJECT.DOMAIN/rest \
      SECURE_REST_ENDPOINT=https://secure-pricing-PROJECT.DOMAIN/rest \ |
      oc create -f -

1. Wait for it to complete. You can follow the build logs with

    oc logs -f bc/ui
    
Access the Demo
---------------
Once all of the above completes, your demo should be running and you can access the UI using http://ui-PROJECT.DOMAIN you can access the secure variant using https://secure-ui-PROJECT.DOMAIN 

Notes
-----
* You can optionally install the 3 templates into OpenShift for use via the GUI using `oc create -f openshift-templates -n openshift`. Once created, you can then deploy the services in your projects using *Add To Project* 


