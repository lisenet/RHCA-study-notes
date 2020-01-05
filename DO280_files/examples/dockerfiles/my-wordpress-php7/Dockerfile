FROM centos:7

RUN yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm && \
    yum -y install yum-utils && yum-config-manager --enable remi-php72 && \
    yum -y --setopt=tsflags=nodocs install httpd php php-mysql php-gd php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-snmp php-soap curl wget unzip && \
    yum clean all && \
    rm -rf /run/httpd/* /tmp/httpd* && \
    cd /var/www/ && wget https://wordpress.org/latest.zip && unzip latest.zip && rm -rf html && mv wordpress html && chown -R apache html

EXPOSE 80

CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND" ]
