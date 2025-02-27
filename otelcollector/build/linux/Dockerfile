FROM --platform=$BUILDPLATFORM mcr.microsoft.com/oss/go/microsoft/golang:1.19 as fluent-bit-builder
WORKDIR /src
COPY ./fluent-bit/src/go.mod ./fluent-bit/src/go.sum .
RUN go version
RUN go mod download
COPY ./fluent-bit/src/ .
RUN apt-get update && apt-get install gcc-aarch64-linux-gnu -y
ARG TARGETOS TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ] ; then CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=c-shared -ldflags '-extldflags=-Wl,-z,now' -o out_appinsights.so . ; else  CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=c-shared -ldflags '-extldflags=-Wl,-z,now' -o out_appinsights.so . ; fi

FROM --platform=$BUILDPLATFORM mcr.microsoft.com/oss/go/microsoft/golang:1.19 as otelcollector-builder
WORKDIR /src
RUN apt-get update && apt-get install gcc-aarch64-linux-gnu -y
RUN go version > goversion.txt
COPY ./opentelemetry-collector-builder/go.mod ./opentelemetry-collector-builder/go.sum ./opentelemetry-collector-builder/
COPY ./prometheusreceiver/go.mod ./prometheusreceiver/go.sum ./prometheusreceiver/
WORKDIR /src/prometheusreceiver
RUN go mod download
WORKDIR /src/opentelemetry-collector-builder
RUN go mod download
COPY ./opentelemetry-collector-builder /src/opentelemetry-collector-builder
COPY ./prometheusreceiver /src/prometheusreceiver
ARG TARGETOS TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ] ; then CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=pie -ldflags '-linkmode external -extldflags=-Wl,-z,now' -o otelcollector . ; else CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=pie -ldflags '-linkmode external -extldflags=-Wl,-z,now' -o otelcollector . ; fi

FROM --platform=$BUILDPLATFORM mcr.microsoft.com/oss/go/microsoft/golang:1.19 as prom-config-validator-builder
WORKDIR /src
RUN apt-get update && apt-get install gcc-aarch64-linux-gnu -y
COPY ./prom-config-validator-builder/go.mod ./prom-config-validator-builder/go.sum ./prom-config-validator-builder/
COPY ./prometheusreceiver/go.mod ./prometheusreceiver/go.sum ./prometheusreceiver/
WORKDIR /src/prometheusreceiver
RUN go version
RUN go mod download
WORKDIR /src/prom-config-validator-builder
RUN go mod download
COPY ./prom-config-validator-builder /src/prom-config-validator-builder
COPY ./prometheusreceiver /src/prometheusreceiver
ARG TARGETOS TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ] ; then CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=pie -ldflags '-linkmode external -extldflags=-Wl,-z,now' -o promconfigvalidator . ; else CGO_ENABLED=1 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -buildmode=pie -ldflags '-linkmode external -extldflags=-Wl,-z,now' -o promconfigvalidator . ; fi

FROM mcr.microsoft.com/cbl-mariner/base/core:2.0 as builder
LABEL description="Azure Monitor Prometheus metrics collector"
LABEL maintainer="ciprometheus@microsoft.com"
ENV OS_TYPE "linux"
ENV tmpdir /opt
# Below is for ContainerInsightsPrometheusCollector-Prod AppInsights Resource
ENV APPLICATIONINSIGHTS_AUTH MWNkYTMxMTItYWY1Ni00ZmNiLWI4MDQtZjg5NDVhYTFjYjMy
ENV TELEMETRY_DISABLED false
# Needed for ME, see https://github.com/microsoft/cpprestsdk/issues/1481
ENV MALLOC_ARENA_MAX=2
COPY ./logrotate/prometheus-collector /etc/logrotate.d/
COPY ./logrotate/logrotate /etc/cron.daily/logrotate
COPY ./logrotate/crontab /etc/crontab
COPY ./scripts/livenessprobe.sh $tmpdir/microsoft/liveness/livenessprobe.sh
COPY ./configmapparser/*.rb $tmpdir/microsoft/configmapparser/
COPY ./configmapparser/default-prom-configs/*.yml $tmpdir/microsoft/otelcollector/default-prom-configs/
COPY ./opentelemetry-collector-builder/collector-config-default.yml ./opentelemetry-collector-builder/collector-config-template.yml ./opentelemetry-collector-builder/PROMETHEUS_VERSION $tmpdir/microsoft/otelcollector/
COPY --from=otelcollector-builder /src/opentelemetry-collector-builder/otelcollector $tmpdir/microsoft/otelcollector/
COPY --from=otelcollector-builder /src/opentelemetry-collector-builder/otelcollector $tmpdir/microsoft/otelcollector/
COPY --from=otelcollector-builder /src/goversion.txt $tmpdir/goversion.txt
COPY --from=prom-config-validator-builder /src/prom-config-validator-builder/promconfigvalidator $tmpdir/
COPY ./scripts/setup.sh ./scripts/main.sh $tmpdir/
COPY ./metricextension/me.config ./metricextension/me_internal.config ./metricextension/me_ds.config ./metricextension/me_ds_internal.config /usr/sbin/
COPY ./telegraf/telegraf-prometheus-collector.conf $tmpdir/telegraf/
COPY ./fluent-bit/fluent-bit.conf ./fluent-bit/fluent-bit-daemonset.conf ./fluent-bit/fluent-bit-parsers.conf $tmpdir/fluent-bit/
COPY --from=fluent-bit-builder /src/out_appinsights.so $tmpdir/fluent-bit/bin/
COPY ./react /static/react
COPY ./LICENSE $tmpdir/microsoft
COPY ./NOTICE $tmpdir/microsoft
COPY ./mdsd/envmdsd $tmpdir/
COPY ./build/linux/rpm-repos/ /etc/yum.repos.d/

ARG TARGETARCH
RUN tdnf repolist --refresh
RUN tdnf install -y wget sudo net-tools cronie vim ruby-devel logrotate procps-ng busybox diffutils curl
RUN mkdir /busybin && busybox --install /busybin
RUN chmod 775 /etc/cron.daily/logrotate
RUN chmod 775 $tmpdir/*.sh; 
RUN sync;
RUN $tmpdir/setup.sh ${TARGETARCH}
# If wanting to run without distroless, uncomment this line and comment everything after
#CMD [ "/opt/main.sh" ]

FROM mcr.microsoft.com/cbl-mariner/distroless/base:2.0
# Below is for ContainerInsightsPrometheusCollector-Prod AppInsights Resource
ENV APPLICATIONINSIGHTS_AUTH MWNkYTMxMTItYWY1Ni00ZmNiLWI4MDQtZjg5NDVhYTFjYjMy
ENV TELEMETRY_DISABLED false
# Needed for ME, see https://github.com/microsoft/cpprestsdk/issues/1481
ENV MALLOC_ARENA_MAX=2
ENV PATH="/busybin:${PATH}"
ENV OS_TYPE "linux"

# files
COPY --from=builder /opt /opt
COPY --from=builder /etc /etc
COPY --from=builder /busybin /busybin
COPY --from=builder /static/react /static/react
COPY --from=builder /usr/sbin/me.config /usr/sbin/me_ds.config /usr/sbin/
COPY --from=builder /var/opt/microsoft /var/opt/microsoft
COPY --from=builder /var/lib/logrotate /var/lib/logrotate
COPY --from=builder /var/spool/cron /var/spool/cron

# executables
COPY --from=builder /usr/sbin/MetricsExtension /usr/sbin/MetricsExtension
COPY --from=builder /usr/bin/ruby /usr/bin/ruby
COPY --from=builder /usr/lib/ruby /usr/lib/ruby
COPY --from=builder /usr/bin/inotifywait /usr/bin/inotifywait
COPY --from=builder /usr/bin/bash /usr/bin/bash
COPY --from=builder /usr/sbin/busybox /usr/sbin/busybox
COPY --from=builder /usr/bin/fluent-bit /usr/bin/fluent-bit
COPY --from=builder /usr/bin/telegraf /usr/bin/telegraf
COPY --from=builder /usr/sbin/crond /usr/sbin/crond
COPY --from=builder /usr/bin/vim /usr/bin/vim
COPY --from=builder /usr/share/vim /usr/share/vim
COPY --from=builder /usr/sbin/mdsd /usr/sbin/mdsd
COPY --from=builder /usr/sbin/logrotate /usr/sbin/logrotate
COPY --from=builder /usr/bin/gzip /usr/bin/
COPY --from=builder /usr/bin/curl /usr/bin/

# bash dependencies
COPY --from=builder /lib/libreadline.so.8 /lib/
COPY --from=builder /usr/lib/libncursesw.so.6 /usr/lib/libtinfo.so.6 /usr/lib/
# inotifywait dependencies
COPY --from=builder /lib/libinotifytools.so.0 /lib/
# crond dependencies
COPY --from=builder /lib/libselinux.so.1 /lib/libpam.so.0 /lib/libc.so.6 /lib/libpcre.so.1 /lib/libaudit.so.1 /lib/libcap-ng.so.0/ /lib/
# vim dependencies
COPY --from=builder /lib/libm.so.6 /lib/libtinfo.so.6 /lib/
# ruby dependencies
COPY --from=builder /usr/lib/libruby.so.3.1 /usr/lib/libz.so.1 /usr/lib/libgmp.so.10 /usr/lib/libcrypt.so.1 /usr/lib/libm.so.6 /usr/lib/
# ruby re2 dependencies
COPY --from=builder /usr/lib/libre2.so.0a /usr/lib/libstdc++.so.6 /usr/lib/libgcc_s.so.1 /usr/lib/libz.so.1 /usr/lib/libgmp.so.10 /usr/lib/libcrypt.so.1 /usr/lib/libm.so.6 /usr/lib/
# metricsextension dependencies
COPY --from=builder /lib/libprotobuf.so.28 /lib/libgrpc++.so.1.42 /lib/libabsl_synchronization.so.2206.0.0 /lib/libboost_filesystem.so.1.76.0 /lib/libboost_coroutine.so.1.76.0 /lib/libboost_thread.so.1.76.0 /lib/libcpprest.so.2.10 /lib/libssl.so.1.1 /lib/libcrypto.so.1.1 /lib/libuuid.so.1 /lib/libstdc++.so.6 /lib/libm.so.6 /lib/libgcc_s.so.1 /lib/libc.so.6 /lib/libz.so.1 /lib64/libgrpc.so.20 /lib/libabsl_raw_hash_set.so.2206.0.0 /lib/libabsl_hash.so.2206.0.0 /lib/libabsl_statusor.so.2206.0.0 /lib/libgpr.so.20 /lib/libupb.so.20 /lib/libabsl_status.so.2206.0.0 /lib/libabsl_time.so.2206.0.0 /lib/libabsl_strings.so.2206.0.0 /lib/libabsl_stacktrace.so.2206.0.0  /lib/libabsl_symbolize.so.2206.0.0 /lib/libabsl_malloc_internal.so.2206.0.0 /lib/libabsl_base.so.2206.0.0 /lib/libabsl_spinlock_wait.so.2206.0.0 /lib/libabsl_raw_logging_internal.so.2206.0.0 /lib/libboost_chrono.so.1.76.0 /lib/libboost_context.so.1.76.0 /lib/libbrotlidec.so.1 /lib/libbrotlienc.so.1 /lib/libcares.so.2 /lib/libaddress_sorting.so.20 /lib/libre2.so.0a /lib/libabsl_cord.so.2206.0.0 /lib/libabsl_str_format_internal.so.2206.0.0 /lib/libabsl_time_zone.so.2206.0.0 /lib/libabsl_city.so.2206.0.0 /lib/libabsl_low_level_hash.so.2206.0.0 /lib/libabsl_cordz_info.so.2206.0.0 /lib/libabsl_strerror.so.2206.0.0 /lib/libabsl_int128.so.2206.0.0 /lib/libabsl_strings_internal.so.2206.0.0 /lib/libabsl_debugging_internal.so.2206.0.0 /lib/libabsl_demangle_internal.so.2206.0.0 /lib/libbrotlicommon.so.1 /lib/libabsl_cord_internal.so.2206.0.0 /lib/libabsl_cordz_functions.so.2206.0.0 /lib/libabsl_cordz_handle.so.2206.0.0 /lib/libabsl_throw_delegate.so.2206.0.0 /lib/libabsl_exponential_biased.so.2206.0.0 /lib/       
# fluent-bit dependencies
COPY --from=builder /lib/libssl.so.1.1 /lib/libcrypto.so.1.1 /lib/libyaml-0.so.2 /lib/libsystemd.so.0 /lib/libm.so.6 /lib/libgcc_s.so.1 /lib/liblzma.so.5 /lib/liblz4.so.1 /lib/libcap.so.2 /lib/libgcrypt.so.20 /lib/libgpg-error.so.0 /lib/
# telegraf dependencies
COPY --from=builder /lib/libc.so.6 /lib/
# mdsd dependencies
COPY --from=builder /usr/lib/libdl.so.2 /usr/lib/librt.so.1 /usr/lib/libpthread.so.0 /usr/lib/libm.so.6 /usr/lib/libstdc++.so.6 /usr/lib/libgcc_s.so.1 /usr/lib/
COPY --from=builder /usr/local/lib/libtcmalloc_minimal.so.4 /usr/local/lib/
# logrotate dependencies
COPY --from=builder /lib/libselinux.so.1 /lib/libpopt.so.0 /lib/libpcre.so.1 /lib/
# curl dependencies
COPY --from=builder /lib/libcurl.so.4 lib/libz.so.1 /lib/libc.so.6 /lib/

RUN [ "/bin/bash", "-c", "chmod 644 /etc/crontab" ]
RUN [ "/bin/bash", "-c", "chown root.root /etc/crontab" ]
RUN [ "/bin/bash", "-c", "chmod 755 /etc/cron.daily/logrotate" ]
RUN [ "/bin/bash", "-c", "chmod 644 /etc/logrotate.d/prometheus-collector" ]
ENTRYPOINT [ "/bin/bash" ]
CMD [ "/opt/main.sh" ]
