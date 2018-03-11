FROM ruby:2.2

RUN apt-get update -qq && apt-get install -y build-essential

ENV APP_HOME /app
RUN mkdir -p $APP_HOME
WORKDIR $APP_HOME

ENV COMMENT_DATABASE_HOST comment_db
ENV COMMENT_DATABASE comments

COPY . .
RUN bundle install

CMD ["puma"]
