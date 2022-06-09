# This file is available under the following license:
# under LGPL 2.1 (LICENSE.TXT) Copyright 2020 Torsten Friebe <tfr@users.sourceforge.net>

FROM tomcat:9-jdk8

LABEL maintainer="deegree TMC <tmc@deegree.org> changed by ..."

# set deegree version
ENV DEEGREE_VERSION=3.4.23
ENV CATALINA_OPTS="-Djavax.xml.transform.TransformerFactory=net.sf.saxon.TransformerFactoryImpl"

EXPOSE 8080

# download deegree webservices webapp
RUN curl https://repo.deegree.org/content/repositories/public/org/deegree/deegree-webservices/${DEEGREE_VERSION}/deegree-webservices-${DEEGREE_VERSION}.war -o /tmp/deegree-webservices.war

RUN unzip -q /tmp/deegree-webservices.war -d /tmp/deegree-webservices/ && \
    chmod -R g+w /tmp/deegree-webservices/

COPY prod/context.xml /tmp/deegree-webservices/META-INF/context.xml
COPY prod/rewrite.config /tmp/deegree-webservices/WEB-INF/rewrite.config

RUN mv /tmp/deegree-webservices/ /usr/local/tomcat/webapps/ && \
    rm -r /tmp/deegree-webservices.war
	
# copy workspace to /root/.deegree
RUN cd /root && \
    mkdir .deegree

COPY workspaces/${WORKSPACE}/ /root/.deegree/
COPY workspaces/webapps.properties /root/.deegree/webapps.properties
COPY prod/main.xml /root/.deegree/deegree_workspace_inspire/services/main.xml

# run tomcat
CMD ["catalina.sh", "run"]