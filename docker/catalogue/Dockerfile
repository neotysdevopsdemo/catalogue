FROM golang:1.7

COPY . src/github.com/neotysdevopsdemo/docker/catalogue
WORKDIR src/github.com/neotysdevopsdemo/docker/catalogue

RUN go get -u github.com/FiloSottile/gvt
RUN gvt restore && \
    CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app src/github.com/neotysdevopsdemo/docker/catalogue/cmd/cataloguesvc

FROM alpine:3.4

ENV	SERVICE_USER=myuser \
	SERVICE_UID=10001 \
	SERVICE_GROUP=mygroup \
	SERVICE_GID=10001

RUN	addgroup -g ${SERVICE_GID} ${SERVICE_GROUP} && \
	adduser -g "${SERVICE_NAME} user" -D -H -G ${SERVICE_GROUP} -s /sbin/nologin -u ${SERVICE_UID} ${SERVICE_USER} && \
	apk add --update libcap

WORKDIR /
COPY --from=0 /app /app
COPY images/ /images/

RUN	chmod +x /app && \
	chown -R ${SERVICE_USER}:${SERVICE_GROUP} /app /images && \
	setcap 'cap_net_bind_service=+ep' /app

USER ${SERVICE_USER}


ARG BUILD_VERSION
ARG COMMIT

LABEL org.label-schema.vendor="neotysdevopsdemo" \
  org.label-schema.version="${BUILD_VERSION}" \
  org.label-schema.name="Socks Shop: Catalogue" \
  org.label-schema.description="REST API for Catalogue service" \
  org.label-schema.url="https://github.com/neotysdevopsdemo/catalogue" \
  org.label-schema.vcs-url="github.com:neotysdevopsdemo/catalogue.git" \
  org.label-schema.vcs-ref="${COMMIT}" \
  org.label-schema.schema-version="1.0"

CMD ["/app", "-port=80"]
EXPOSE 80
