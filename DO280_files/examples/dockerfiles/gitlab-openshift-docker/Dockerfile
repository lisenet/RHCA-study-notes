FROM centos:7
MAINTAINER Erik Jacobs <erikmjacobs@gmail.com>

ENV RUBY_VERSION="2.1.9" \
    RUBY_URL="https://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.9.tar.gz" \
    RUBYGEMS_VERSION="2.6.4" \
    RUBYGEMS_URL="https://rubygems.org/rubygems/rubygems-2.6.4.tgz" \
    GITLAB_VERSION="8.8.1" \
    BUILD_PKGS="gcc-c++ automake cmake make autoconf which patch" \
    REMOVE_PKGS=0

# install prereqs and compilers
RUN yum clean all && \
    yum -y install centos-release-scl && \
    export INSTALL_PKGS="curl-devel openssl-devel readline-devel libffi-devel libyaml-devel \
    zlib-devel libxslt-devel libxml2-devel libicu-devel iconv-devel \
    mysql-libs mysql-devel postgresql-devel sqlite-devel nodejs010 git gettext"  && \
    yum install -y --setopt=tsflags=nodocs \
    $INSTALL_PKGS $BUILD_PKGS && \
    yum clean all && \
    cd /usr/local/src && \
    curl -L $RUBY_URL -o ruby-$RUBY_VERSION.tgz && \
    tar -xzvf ruby-$RUBY_VERSION.tgz && \
    rm -f ruby-$RUBY_VERSION.tgz

# compile ruby
RUN cd /usr/local/src/ruby-$RUBY_VERSION && \
    ./configure && make && make install

# install rubygems and bundler
RUN cd /usr/local/src && \
    curl -L $RUBYGEMS_URL -o rubygems-$RUBYGEMS_VERSION.tgz && \
    tar -xzvf rubygems-$RUBYGEMS_VERSION.tgz && \
    rm -f rubygems-$RUBYGEMS_VERSION.tgz && \
    cd rubygems-$RUBYGEMS_VERSION && \
    ruby setup.rb && \
    gem install bundler --no-ri --no-rdoc

# will need to add gitlab gemfile for bundle
# or will need to separate gem installation
# unless nodejs fixes deps issues

# unpack gitlab and install its gems
RUN cd /usr/local/src && \
    curl -L https://gitlab.com/gitlab-org/gitlab-ce/repository/archive.tar.gz?ref=v$GITLAB_VERSION \
      -o gitlab.tgz && \
    tar -xzvf gitlab.tgz && \
    rm -f gitlab.tgz && \
    mv $(ls -1 | grep gitlab) gitlab-ce-v$GITLAB_VERSION && \
    cd /usr/local/src/gitlab-ce-v$GITLAB_VERSION && \
    bundle install --without development test mysql aws kerberos

# uninstall compilers if asked/possible to slightly reduce space and
# make a more "secure" image, in some ways
RUN if [ $REMOVE_PKGS -eq 1 ]; then yum -y remove $BUILD_PKGS && \
    yum clean all; fi

# add unicorn config and correctly sub in path(s)
ADD config /tmp/config
RUN cat /tmp/config/unicorn.rb | envsubst > /usr/local/src/gitlab-ce-v$GITLAB_VERSION/config/unicorn.rb && \
    cp /tmp/config/database.yml /usr/local/src/gitlab-ce-v$GITLAB_VERSION/config && \
    rm -rf /tmp/config

# ultimately will need to run via SCL
# cd /usr/local/src/gitlab-ce-v$GITLAB_VERSION && scl enable nodejs010 v8314 -- bundle exec unicorn_rails -c config/unicorn.rb -E production
