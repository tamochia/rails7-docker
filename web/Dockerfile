FROM docker.io/library/ruby:3.3

ARG RAILS_APP_PATH RUN_USER RUN_UID

WORKDIR $RAILS_APP_PATH

ENV TZ=Asia/Tokyo RUN_UID=$RUN_UID RUN_USER=$RUN_USER

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential default-libmysqlclient-dev default-mysql-client pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .

RUN useradd -u $RUN_UID $RUN_USER --create-home --shell /bin/bash && \
    chown -R $RUN_USER:$RUN_USER .
USER $RUN_USER:$RUN_USER
 
CMD ["/bin/bash"]

