FROM centos:7
ENV container docker
MAINTAINER ManageIQ https://github.com/ManageIQ/manageiq-appliance-build
ARG REF=master

# Set ENV, LANG only needed if building with docker-1.8
ENV LANG en_US.UTF-8
ENV TERM xterm
ENV RUBY_GEMS_ROOT /opt/rubies/ruby-2.3.1/lib/ruby/gems/2.3.0
ENV APP_ROOT /var/www/miq/vmdb
ENV APP_ROOT_PERSISTENT /persistent
ENV APPLIANCE_ROOT /opt/manageiq/manageiq-appliance
ENV SUI_ROOT /opt/manageiq/manageiq-ui-service
ENV CONTAINER_SCRIPTS_ROOT /opt/manageiq/container-scripts
ENV IMAGE_VERSION ${REF}

## Install EPEL repo, yum necessary packages for the build without docs, clean all caches
RUN yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && \
    yum -y install centos-release-scl-rh && \
    yum -y install --setopt=tsflags=nodocs \
                   bison                   \
                   bzip2                   \
                   cmake                   \
                   file                    \
                   gcc-c++                 \
                   git                     \
                   libcurl-devel           \
                   libffi-devel            \
                   libtool                 \
                   libxml2-devel           \
                   libxslt-devel           \
                   libyaml-devel           \
                   make                    \
                   net-tools               \
                   nodejs                  \
                   openssl-devel           \
                   openscap-scanner        \
                   patch                   \
                   rh-postgresql95-postgresql-libs \
                   rh-postgresql95-postgresql-devel  \
                   readline-devel          \
                   sqlite-devel            \
                   sysvinit-tools          \
                   which                   \
                   httpd                   \
                   mod_ssl                 \
                   mod_auth_kerb           \
                   mod_authnz_pam          \
                   mod_intercept_form_submit \
                   mod_lookup_identity     \
                   initscripts             \
                   npm                     \
                   chrony                  \
                   psmisc                  \
                   lvm2                    \
                   openldap-clients        \
                   gdbm-devel              \
                   cronie                  \
                   logrotate               \
                   &&                      \
    yum clean all

## Systemd cleanup base image
RUN (cd /lib/systemd/system/sysinit.target.wants && for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -vf $i; done) && \
    rm -vf /lib/systemd/system/multi-user.target.wants/* && \
    rm -vf /etc/systemd/system/*.wants/* && \
    rm -vf /lib/systemd/system/local-fs.target.wants/* && \
    rm -vf /lib/systemd/system/sockets.target.wants/*udev* && \
    rm -vf /lib/systemd/system/sockets.target.wants/*initctl* && \
    rm -vf /lib/systemd/system/basic.target.wants/* && \
    rm -vf /lib/systemd/system/anaconda.target.wants/*

# Download chruby and chruby-install, install, setup environment, clean all
RUN curl -sL https://github.com/postmodern/chruby/archive/v0.3.9.tar.gz | tar xz && \
    cd chruby-0.3.9 && \
    make install && \
    scripts/setup.sh && \
    echo "gem: --no-ri --no-rdoc --no-document" > ~/.gemrc && \
    echo "source /usr/local/share/chruby/chruby.sh" >> ~/.bashrc && \
    curl -sL https://github.com/postmodern/ruby-install/archive/v0.6.0.tar.gz | tar xz && \
    cd ruby-install-0.6.0 && \
    make install && \
    ruby-install ruby 2.3.1 -- --disable-install-doc && \
    echo "chruby ruby-2.3.1" >> ~/.bash_profile && \
    rm -rf /chruby-* && \
    rm -rf /usr/local/src/* && \
    yum clean all

## GIT clone manageiq-appliance and service UI repo (SUI)
RUN mkdir -p ${APP_ROOT} && \
    mkdir -p ${APPLIANCE_ROOT} && \
    mkdir -p ${SUI_ROOT} && \
    ln -vs ${APP_ROOT} /opt/manageiq/manageiq && \
    curl -L https://github.com/ManageIQ/manageiq-appliance/tarball/${REF} | tar vxz -C ${APPLIANCE_ROOT} --strip 1 && \
    curl -L https://github.com/ManageIQ/manageiq-ui-service/tarball/${REF} | tar vxz -C ${SUI_ROOT} --strip 1

## Add ManageIQ source from local directory (dockerfile development) or from Github (official build)
#ADD . ${APP_ROOT}
RUN curl -L https://github.com/ManageIQ/manageiq/tarball/${REF} | tar vxz -C ${APP_ROOT} --strip 1

## Setup environment
RUN ${APPLIANCE_ROOT}/setup && \
    echo "export PATH=\$PATH:/opt/rubies/ruby-2.3.1/bin" >> /etc/default/evm && \
    mkdir ${APP_ROOT}/log/apache && \
    mkdir ${APP_ROOT_PERSISTENT} && \
    mkdir -p ${CONTAINER_SCRIPTS_ROOT} && \
    mv /etc/httpd/conf.d/ssl.conf{,.orig} && \
    echo "# This file intentionally left blank. ManageIQ maintains its own SSL configuration" > /etc/httpd/conf.d/ssl.conf && \
    cp ${APP_ROOT}/config/cable.yml.sample ${APP_ROOT}/config/cable.yml && \
    echo "export APP_ROOT=${APP_ROOT}" >> /etc/default/evm && \
    echo "export CONTAINER=true" >> /etc/default/evm

## Change workdir to application root, build/install gems
WORKDIR ${APP_ROOT}
RUN source /etc/default/evm && \
    export RAILS_USE_MEMORY_STORE="true" && \
    npm install bower yarn -g && \
    gem install bundler --conservative && \
    bower install --allow-root -F --silent --config.analytics=false && \
    bundle install && \
    bin/rails log:clear tmp:clear && \
    rake evm:compile_assets && \
    rake evm:compile_sti_loader && \
    # Cleanup install artifacts
    npm cache clean && \
    bower cache clean && \
    find ${RUBY_GEMS_ROOT}/gems/ -name .git | xargs rm -rvf && \
    find ${RUBY_GEMS_ROOT}/gems/ | grep "\.s\?o$" | xargs rm -rvf && \
    rm -rvf ${RUBY_GEMS_ROOT}/gems/rugged-*/vendor/libgit2/build && \
    rm -rvf ${RUBY_GEMS_ROOT}/cache/* && \
    rm -rvf /root/.bundle/cache && \
    rm -rvf ${APP_ROOT}/tmp/cache/assets && \
    rm -rvf ${APP_ROOT}/log/*

## Build SUI
RUN source /etc/default/evm && \
    cd ${SUI_ROOT} && \
    yarn install && \
    yarn run build

## Copy OpenShift and appliance-initialize scripts, add systemd service unit file
COPY docker-assets/entrypoint /usr/bin
COPY docker-assets/container.data.persist /
COPY docker-assets/appliance-initialize.service /usr/lib/systemd/system
COPY docker-assets/appliance-initialize.sh /bin
ADD  docker-assets/container-scripts ${CONTAINER_SCRIPTS_ROOT}

## Scripts symlinks
RUN ln -s /var/www/miq/vmdb/docker-assets/docker_initdb /usr/bin

## Enable services on systemd
RUN systemctl enable appliance-initialize evmserverd evminit evm-watchdog miqvmstat miqtop crond

## Expose required container ports
EXPOSE 80 443

## Atomic Labels
# The UNINSTALL label by DEFAULT will attempt to delete a container (rm) and image (rmi) if the container NAME is the same as the actual IMAGE
# NAME is set via -n flag to ALL atomic commands (install,run,stop,uninstall)
LABEL name="manageiq" \
      vendor="ManageIQ" \
      version="Master" \
      release=${REF} \
      architecture="x86_64" \
      url="http://manageiq.org/" \
      summary="ManageIQ appliance image" \
      description="ManageIQ is a management and automation platform for virtual, private, and hybrid cloud infrastructures." \
      INSTALL='docker run -ti --privileged \
                --name ${NAME}_volume \
                --entrypoint /usr/bin/docker_initdb \
                $IMAGE' \
      RUN='docker run -di --privileged \
            --name ${NAME}_run \
            -v /etc/localtime:/etc/localtime:ro \
            --volumes-from ${NAME}_volume \
            -p 80:80 \
            -p 443:443 \
            $IMAGE' \
      STOP='docker stop ${NAME}_run && echo "Container ${NAME}_run has been stopped"' \
      UNINSTALL='docker rm -v ${NAME}_volume ${NAME}_run && echo "Uninstallation complete"'

## OpenShift labels
LABEL io.k8s.description="ManageIQ is a management and automation platform for virtual, private, and hybrid cloud infrastructures." \
      io.k8s.display-name="ManageIQ" \
      io.openshift.expose-services="443:https" \
      io.openshift.tags="ManageIQ,miq,manageiq"

## Call systemd to bring up system
ENTRYPOINT ["entrypoint"]
CMD [ "/usr/sbin/init" ]
