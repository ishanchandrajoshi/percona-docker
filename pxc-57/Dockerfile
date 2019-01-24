FROM centos:6
MAINTAINER Percona Development <info@percona.com>

ARG MYSQL_USER=mysql
ARG MYSQL_GROUP=mysql
ARG LABEL_PLATFORM
ARG WITH_AWS=false
ARG WITH_PAM_LDAP=false

ENV MYSQL_USER $MYSQL_USER

RUN groupadd -g 65534 nogroup \
    && usermod -u 99 -g 99 -c "Nobody user for RHEL based system" nobody \
    && echo "nobody:x:65534:65534:Nobody user for Debian based system:/nonexistent:/sbin/nologin" >> /etc/passwd \
    && usermod -a -G users nobody

#RUN sh -c "getent group ${MYSQL_GROUP} || groupadd -g 65534 ${MYSQL_GROUP}"

#RUN usermod -u 99 nobody -l copynobody
#RUN sh -c "getent passwd ${MYSQL_USER} || useradd -u 65534 -r -g $(getent group ${MYSQL_GROUP} | awk -F: '{print $3}') -s /sbin/nologin -c 'Default Application User' ${MYSQL_USER}"

RUN yum update -y -q && yum install -y -q yum-utils \
		ca-certificates wget

RUN wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm \
  && rpm -Uh epel-release-6-8.noarch.rpm

RUN wget http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el6/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm \
  && rpm -Uh rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm

RUN yum install -y -q libev socat pwgen curl

RUN yum-config-manager --add-repo https://repo.percona.com/yum/release/centos/latest/os/x86_64/ \
  && curl -O https://www.percona.com/downloads/RPM-GPG-KEY-percona \
  && mv RPM-GPG-KEY-percona /etc/pki/rpm-gpg/ \
  && rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-percona

RUN     yum update -y -q \
	&& yum install -y -q -- Percona-XtraDB-Cluster-57-5.7.21-29.26.1.el6 \
        && ( ( ${WITH_AWS} \
               && yum install -y -q python34 python34-devel libyaml-devel \
               && curl -O https://bootstrap.pypa.io/get-pip.py \
               && python3 get-pip.py \
               && pip3 install awscli --upgrade \
             ) || true \
        ) \
        && ( ( ${WITH_PAM_LDAP} \
               && yum install -y -q pam_ldap \
             ) || true \
        ) \
# comment out any "user" entires in the MySQL config ("docker-entrypoint.sh" or "--user" will handle user switching)
	&& sed -ri 's/^user/#&/' /etc/my.cnf \
# purge and re-create /var/lib/mysql with appropriate ownership
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld /etc/mysql/conf.d/ /var/log/mysql \
	&& chown -R ${MYSQL_USER}:${MYSQL_GROUP} /var/lib/mysql /var/run/mysqld /var/log/mysql /etc/nsswitch.conf /etc/pam_ldap.conf /etc/pam.d \
        && chmod g+rwx -R /var/lib/mysql /var/run/mysqld /var/log/mysql /etc/nsswitch.conf /etc/pam_ldap.conf /etc/pam.d \
        && touch /var/log/mysql/error.log \
        && find /var/log/mysql* -exec sh -c "chown -v ${MYSQL_USER}:${MYSQL_GROUP} {}; chmod -v u+rw,g+r,o-rwx {}" \; \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	&& chmod 777 /var/run/mysqld

RUN sed -ri 's/^bind-address/#&/' /etc/my.cnf 

#	&& echo 'skip-host-cache\nskip-name-resolve' | awk '{ print } $1 == "[mysqld]" && c == 0 { c = 1; system("cat") }' /etc/mysql/my.cnf > /tmp/my.cnf \
#	&& mv /tmp/my.cnf /etc/mysql/my.cnf


VOLUME ["/var/lib/mysql", "/var/log/mysql"]

RUN sed -ri 's/^log-error/#&/' /etc/my.cnf

ADD node.cnf /etc/mysql/conf.d/node.cnf
RUN echo '!includedir /etc/mysql/conf.d/' >> /etc/my.cnf
RUN chown -R ${MYSQL_USER}:${MYSQL_GROUP} /etc/mysql
RUN chmod g+rwx -R /etc/mysql

COPY entrypoint.sh /entrypoint.sh
COPY jq /usr/bin/jq
COPY clustercheckcron /usr/bin/clustercheckcron

RUN chmod a+x /usr/bin/jq
RUN chmod a+x /usr/bin/clustercheckcron

EXPOSE 3306 4567 4568 4444

LABEL vendor=Percona
LABEL com.percona.package="Percona XtraDB Cluster"
LABEL com.percona.version="5.7"
LABEL com.percona.platform="${LABEL_PLATFORM}"

ENTRYPOINT ["/entrypoint.sh", "${MYSQL_USER}"]

USER ${MYSQL_USER}:${MYSQL_GROUP} 
CMD [""]
