FROM ruby:2.3.1-alpine

MAINTAINER cloudnil <cloudnil@126.com>

ENV TZ 'Asia/Shanghai'

RUN apk upgrade --no-cache && \
    apk add --no-cache bash tzdata nodejs py-pygments git gcc musl-dev make && \
    git clone https://github.com/cloudnil/cloudnil.github.io.git /root/cloudnil.github.io && \
    cd /root/cloudnil.github.io && \
    gem install --no-document bundle && \
    bundle install && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del gcc musl-dev make && \
    rm -rf /var/cache/apk/*

COPY cron/15min/* /etc/periodic/15min
COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s /usr/local/bin/docker-entrypoint.sh /docker-entrypoint.sh

WORKDIR /root

EXPOSE 80
CMD ["docker-entrypoint.sh"]