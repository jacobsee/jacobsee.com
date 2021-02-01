---
title: "Integrating Red Hat Single Sign-On with X.509 Client Certificates on OpenShift"
description: "So that you don't have to struggle like I did!"
date: 2019-11-28
---

> Commands in this post are written using `podman`. If you use `docker` instead, simply replace every instance of `podman` with `docker`. The syntax of the commands is interchangeable.

If you use `podman inspect` to dive into the `sso72-openshift` container image provided with Red Hat OpenShift, you'll find an interesting phrase in a label called "usage" - "This image is very generic and does not serve a single use case. Use it as a base to build your own images." I thought this was strange, as I've never had to base a secondary image off of the SSO image to gain value from it. It's always worked fine out of the box... until I tried to use it for validating X.509 client certificates.

For starters, let's make sure that we understand how client validation using X.509 differs from simply using a username and password on a protocol level. When you log in to a website with a username and password, your browser first negotiates a secure TLS connection to the server, and then uses HTTP-style requests over that TLS connection to communicate with it - including POSTing the credentials to the server when you click the "submit" button. Step one, secure the connection; step two, post credentials using HTTP methods.

Authenticating using client certificates _does not_ follow this same flow. Client certificates, also called Mutual TLS, are shared in the _first_ step - when the browser initially tries to negotiate a secure connection to the server. This means that in order to set up client certificate validation, what we actually need to configure is _how SSO negotiates TLS_, in addition to how those client-provided certificates map to user accounts (which we'll get to later).

If you spend any time with the SSO images provided by Red Hat, you'll see that there is a startup script which modifies the configuration for WildFly based on its environment at runtime. Convenient for some things, but we're going to need to modify that initial configuration in a new way right now. Let's start by grabbing the unmodified SSO configuration file. Run `podman run -it --rm sso72-openshift /bin/bash` to open a shell into the container. Notice how we've overridden the `COMMAND` of the container with `/bin/bash`, so the script which would overwrite our config file is _not_ run. You will be placed in a new shell, with a prompt that looks something like `[jboss@77d1a9f847bd ~] $`. Take note of that random-looking string after the `@`, in this case `77d1a9f847bd`. This is the ID of the container we just started. Leave this shell running in the background and open a new one. In this new shell, run `podman cp 77d1a9f847bd:/opt/eap/standalone/configuration/standalone-openshift.xml standalone-openshift.xml`, making sure to replace my container ID with your own. You can close the original terminal and remove the container now - we've just copied out the WildFly configuration that we need!

Open the `standalone-openshift.xml` from your current directory in your favorite editor, and search for `HTTPS`. You will find one result, a line (537 in the image I used) with a placeholder value of `<!-- ##HTTPS_CONNECTOR## -->` on it. We're not going to need this, so go ahead and delete this line. In its place, add the actual configuration for an HTTPS listener, which looks like this:

```xml
<https-listener name="default-ssl" socket-binding="https" security-realm="ssl-realm" verify-client="REQUESTED" />
```

The important bit here is that we've defined an HTTPS listener with `verify-client` enabled. That flag enables Mutual TLS as described above.

Now, there's something I'm not going to cover in this post that you'll need to figure out on your own. I'm not trying to withhold information, but this bit will be different in every individual case. The thing is this: Who is the certificate authority (CA) that signs the client certificates that we want to trust? Obviously we don't want to trust _every_ certificate presented to us, or somebody could generate a self-signed certificate and log in as anyone! For the purposes of this post, we'll say that you have a certificate file (or multiple files for a chain) that represents the CA whose signature you'd like to trust for validating client certificates.

Once you have this CA established, we'll need to turn it into a JKS truststore. You can use `keytool` for this if it's installed on your system, but I'm going to recommend that you give [KeyStore Explorer](https://keystore-explorer.org) a try. It's as simple as creating a new keystore, adding certificates, and saving it as a JKS file. You will be prompted to enter a password for the keystore - make sure you don't lose it.

Now that we have our truststore saved as a JKS file, we need to configure SSO to use it. Open that same `standalone-openshift.xml` file that we modified earlier - we're going to make another change. Towards the beginning (line 31 for me), you will find a block defining security realms. It looks something like this:

```xml
<security-realms>
    <security-realm name="ManagementRealm">
        <authentication>
            <local default-user="$local" skip-group-loading="true"/>
            <properties path="mgmt-users.properties" relative-to="jboss.server.config.dir"/>
        </authentication>
        <authorization map-groups-to-roles="false">
            <properties path="mgmt-groups.properties" relative-to="jboss.server.config.dir"/>
        </authorization>
    </security-realm>
    <security-realm name="ApplicationRealm">
        <authentication>
            <local default-user="$local" allowed-users="*" skip-group-loading="true"/>
            <properties path="application-users.properties" relative-to="jboss.server.config.dir"/>
        </authentication>
        <authorization>
            <properties path="application-roles.properties" relative-to="jboss.server.config.dir"/>
        </authorization>
        <!-- ##SSL## -->
    </security-realm>
</security-realms>
```

We need to make a change here to tell SSO to verify client certificates using our newly-generated truststore. Under the security realm `ApplicationRealm`, in the `authentication` subsection, add `<truststore path="/home/jboss/your-truststore-name.jks" keystore-password="<your truststore password>"/>` (obviously with your own parameters filled in) so that you end up with:

```xml
<security-realms>
    <security-realm name="ManagementRealm">
        <authentication>
            <local default-user="$local" skip-group-loading="true"/>
            <properties path="mgmt-users.properties" relative-to="jboss.server.config.dir"/>
        </authentication>
        <authorization map-groups-to-roles="false">
            <properties path="mgmt-groups.properties" relative-to="jboss.server.config.dir"/>
        </authorization>
    </security-realm>
    <security-realm name="ApplicationRealm">
        <authentication>
            <local default-user="$local" allowed-users="*" skip-group-loading="true"/>
            <properties path="application-users.properties" relative-to="jboss.server.config.dir"/>
            <truststore path="/home/jboss/your-truststore-name.jks" keystore-password="<your truststore password>"/>
        </authentication>
        <authorization>
            <properties path="application-roles.properties" relative-to="jboss.server.config.dir"/>
        </authorization>
        <!-- ##SSL## -->
    </security-realm>
</security-realms>
```

That's all the changes we need to make to this configuration file! Now, let's build a new container image with these changes. Take the directory that you're currently working in, make a git repository, and commit the `standalone-openshift.xml` and truststore JKS files. Push the repository to GitHub, GitLab, or similar.

Make a new file called `build.yml` and add the following to it:

```yaml
kind: Template
apiVersion: v1
metadata:
  name: "${NAME}"
  annotations:
    openshift.io/display-name: "${NAME} Build"
    description: Build config to create an app image.
    iconClass: fa-cube
    tags: apps
objects:
- apiVersion: v1
  kind: BuildConfig
  metadata:
    labels:
      build: "${NAME}"
    name: "${NAME}"
  spec:
    nodeSelector:
    output:
      to:
        kind: ImageStreamTag
        name: "${NAME}:latest"
    postCommit: {}
    resources: {}
    runPolicy: Serial
    source:
      dockerfile: |
          FROM registry.access.redhat.com/redhat-sso-7/sso73-openshift
          ADD your-truststore-name.jks your-truststore-name.jks
          ADD standalone-openshift.xml /opt/eap/standalone/configuration/standalone-openshift.xml
      git:
        ref: "${SOURCE_REPOSITORY_REF}"
        uri: "${SOURCE_REPOSITORY_URL}"
      type: Git
    strategy:
      dockerStrategy: {}
  status:
    lastVersion: 1
- apiVersion: v1
  kind: ImageStream
  metadata:
    labels:
      build: "${NAME}"
    name: "${NAME}"
  spec: {}
parameters:
- name: NAME
  displayName: Name
  description: The name assigned to all objects and the resulting imagestream.
  required: true
- name: SOURCE_REPOSITORY_URL
  displayName:
  description:
  required: true
- name: SOURCE_REPOSITORY_REF
  displayName:
  description:
  required: true
labels:
  template: app-build-template
```

Apply this file to your OpenShift cluster with `oc process -f build.yml NAME=my-sso SOURCE_REPOSITORY_URL=<your-repo-url> SOURCE_REPOSITORY_REF=<your-branch-name> | oc apply -f -`. You should then be able to `oc start-build my-sso` and build this image directly in the cluster.

All that's left is to configure an deployment of this newly-baked image! OpenShift has some useful templates for us to start with. You can fetch one of these with `oc get template sso72-postgresql-persistent -n openshift -o yaml`, and replace the `image: ${APPLICATION_NAME}` with the name of the `ImageStream` you created above (or keep the variable and simply make sure that you name your application the same thing as the `ImageStream`). Alternatively, if you just want to try this out quickly, the catalog in the OpenShift web console does a fine job of deploying SSO with point-and-click ease, after which you can simply edit your new image into the deployment!

![](/images/rh-sso-ocp.png)