# requires secret: hsm-server-secret (file that contains password for HSM_USER)
  
# requires ARG: GW_IMAGE (example caapim/gateway:10.1.00)
# requires ARG: CLIENT_TAR_FILENAME (example 610-000401-003_SW_Linux_Luna_Minimal_Client_V10.3.0_RevA.tar)   
# requires ARG: CLIENT_PATCH_TAR_FILENAME (example 630-000522-001_Sw_Patch_jsp_fix_for_jdk11_UC10.3.0_Custom_Release.tar) 
# requires ARG: HSM_USER
# requires ARG: HSM_SERVER
# requires ARG: CERT_NAME
# requires ARG: PARTITION_NAME
  
ARG GW_IMAGE=caapim/gateway:10.1.00
ARG CENTOS_IMAGE=centos:centos7
  
FROM ${CENTOS_IMAGE} AS luna-min-client
  
ARG CLIENT_TAR_FILENAME
ARG CLIENT_PATCH_TAR_FILENAME
ARG HSM_SERVER
ARG HSM_USER
ARG CERT_NAME
ARG PARTITION_NAME
  
RUN yum update -y \
    && yum install -y \
      sed \
      sshpass \
      openssh-clients \
    && yum clean all
  
ENV LUNA_DIR /home/luna-docker
  
RUN mkdir -p $LUNA_DIR/config/certs
  
# Copy the Luna Minimal Client tarball to $LUNA_DIR
COPY ${CLIENT_TAR_FILENAME} $LUNA_DIR/.
 
# Untar the Luna Minimal Client tarball
RUN tar -xf $LUNA_DIR/${CLIENT_TAR_FILENAME} --strip 1 -C $LUNA_DIR; rm -f $LUNA_DIR/${CLIENT_TAR_FILENAME}
 
# Copy the Chrystoki.conf file from the Minimal Client directory to $LUNA_DIR/config
# ENV MIN_CLIENT_DIR $LUNA_DIR/${CLIENT_TAR_FILENAME}
WORKDIR $LUNA_DIR
RUN cp Chrystoki-template.conf $LUNA_DIR/config/Chrystoki.conf
 
# Define the following environment variable
ENV ChrystokiConfigurationPath $LUNA_DIR/config
 
# Update the Chrystoki.conf file paths so the tools work as expoected
RUN ./bin/64/configurator setValue -s Chrystoki2 -e LibUNIX -v $LUNA_DIR/libs/64/libCryptoki2.so \
  && ./bin/64/configurator setValue -s Chrystoki2 -e LibUNIX64 -v $LUNA_DIR/libs/64/libCryptoki2_64.so \
  && ./bin/64/configurator setValue -s Misc -e ToolsDir -v $LUNA_DIR/bin/64 \
  && ./bin/64/configurator setValue -s "LunaSA Client" -e SSLConfigFile -v $LUNA_DIR/openssl.cnf \
  && ./bin/64/configurator setValue -s "LunaSA Client" -e ClientPrivKeyFile -v $LUNA_DIR/config/certs/dockerlunaclientKey.pem \
  && ./bin/64/configurator setValue -s "LunaSA Client" -e ClientCertFile -v $LUNA_DIR/config/certs/dockerlunaclient.pem \
  && ./bin/64/configurator setValue -s "LunaSA Client" -e ServerCAFile -v $LUNA_DIR/config/certs/CAFile.pem  \
  && ./bin/64/configurator setValue -s "Secure Trusted Channel" -e ClientTokenLib -v $LUNA_DIR/libs/64/libSoftToken.so \
  && ./bin/64/configurator setValue -s "Secure Trusted Channel" -e SoftTokenDir -v $LUNA_DIR/config/stc/token \
  && ./bin/64/configurator setValue -s "Secure Trusted Channel" -e ClientIdentitiesDir -v $LUNA_DIR/config/stc/client_identities \
  && ./bin/64/configurator setValue -s "Secure Trusted Channel" -e PartitionIdentitiesDir -v $LUNA_DIR/config/stc/partition_identities

# copy the hsm appliance server certificate
RUN --mount=type=secret,id=hsm-server-secret,dst=/.hsm-server-secret sshpass -f /.hsm-server-secret scp -oStrictHostKeyChecking=no ${HSM_USER}@${HSM_SERVER}:server.pem $LUNA_DIR/config/certs

# Register the appliance server certificate with the Client
RUN ./bin/64/vtl addServer -c config/certs/server.pem -n ${HSM_SERVER}

# Following is only needed if you need to generate client key/certificate.
# Alternatively, you can use an existing ones and copy the private key/certificate to $LUNA_DIR/config/certs)
# Create a Luna HSM Client certificate for the Docker container ($MIN_CLIENT_DIR/bin/64/vtl createCert -n <cert name>)
RUN ./bin/64/vtl createCert -n ${CERT_NAME}
# Copy the client certificate to the SafeNet Luna Network HSM appliance
RUN --mount=type=secret,id=hsm-server-secret,dst=/.hsm-server-secret sshpass -f /.hsm-server-secret scp -oStrictHostKeyChecking=no config/certs/${CERT_NAME}.pem ${HSM_USER}@${HSM_SERVER}:

# Register client
RUN --mount=type=secret,id=hsm-server-secret,dst=/.hsm-server-secret sshpass -f /.hsm-server-secret ssh ${HSM_USER}@${HSM_SERVER} "client register -client ${CERT_NAME} -hostname ${CERT_NAME}"
# Assign client to partition
RUN --mount=type=secret,id=hsm-server-secret,dst=/.hsm-server-secret sshpass -f /.hsm-server-secret ssh ${HSM_USER}@${HSM_SERVER} "client assignpartition -client ${CERT_NAME} -partition ${PARTITION_NAME}"
# Disable ip check
RUN --mount=type=secret,id=hsm-server-secret,dst=/.hsm-server-secret sshpass -f /.hsm-server-secret ssh ${HSM_USER}@${HSM_SERVER} "ntls ipcheck disable"
  
RUN sed -i -e 's#'$LUNA_DIR'/config#/usr/local/luna/config#g' -e 's#'$LUNA_DIR'#/usr/local/luna#g' $ChrystokiConfigurationPath/Chrystoki.conf

# Copy and extract JDK11 patch
RUN mkdir -p /tmp/patch
 
COPY ${CLIENT_PATCH_TAR_FILENAME} /tmp/patch/.
 
# Extract patch to /tmp/patch
RUN tar -xf /tmp/patch/${CLIENT_PATCH_TAR_FILENAME} --strip 1 -C /tmp/patch; rm -f /tmp/patch/${CLIENT_PATCH_TAR_FILENAME}

#
# Build new GW image with Luna client
#
FROM ${GW_IMAGE} AS gateway

ARG CLIENT_TAR_FILENAME

# Add the luna client
USER root
COPY ${CLIENT_TAR_FILENAME} /tmp
ENV LUNA_DIR /usr/local/luna
RUN mkdir -p ${LUNA_DIR} \
  && tar xvf /tmp/${CLIENT_TAR_FILENAME} --strip 1 -C ${LUNA_DIR} \
  && rm -f /tmp/${CLIENT_TAR_FILENAME}

COPY --from=luna-min-client "/home/luna-docker/config" "${LUNA_DIR}/config"
ENV ChrystokiConfigurationPath /usr/local/luna/config
ENV PATH="${LUNA_DIR}/bin/64:${PATH}"


# For non patched version, copy required jsp files from ${MIN_CLIENT_dir} directly
#
COPY --from=luna-min-client /tmp/patch/LunaProvider.jar /opt/SecureSpan/Gateway/runtime/lib/ext/
COPY --from=luna-min-client /tmp/patch/linux/64/libLunaAPI.so /opt/SecureSpan/Gateway/runtime/lib/ext/

#
# Update security provider order as required by Luna client.  This can be done either here or in the helm chart.
# For this example, it is done in the helmchart so that the provider order can be reverted if there is a need
# to disable Luna HSM.  Uncomment if provider order is to be set at image creation time.
#RUN echo "security.provider.1=SUN" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.2=SunJCE" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.3=SunJGSS" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.4=SunSASL" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.5=XMLDSig" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.6=SunPCSC" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.7=JdkLDAP" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.8=JdkSASL" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.9=SunPKCS11" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.10=com.safenetinc.luna.provider.LunaProvider" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.11=SunRsaSign" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.12=SunEC" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "security.provider.13=SunJSSE" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security

#  
# Override the default Luna configuration.  Like above, for this example, it is configured in Helm. 
# Uncomment if this configuration is to be set at image creation time. 
#RUN echo "com.safenetinc.luna.provider.createExtractablePrivateKeys=true" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security \
#    && echo "com.safenetinc.luna.provider.createExtractableSecretKeys=true" >> /opt/SecureSpan/Gateway/runtime/etc/ssg.security

# Make sure to switch back to the entrypoint user at the end.
USER ${ENTRYPOINT_UID}
