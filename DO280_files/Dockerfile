FROM centos:7

RUN yum -y --setopt=tsflags=nodocs install httpd php && \
    yum clean all && \
    rm -rf /run/httpd/* /tmp/httpd* && \
    echo "Apache in a Docker container" > /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND" ]
