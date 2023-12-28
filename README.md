Ruby with Python
===

The `ruby` slim with additional python for [pycall.rb](https://github.com/mrkn/pycall.rb/tree/master)

## Example

```dockerfile
# Specify Version
ARG RUBY_VERSION=3.3.0
ARG PYTHON_VERSION=3.12.1

# Install Ruby Gems
FROM ruby:${RUBY_VERSION} AS gem

RUN mkdir -p /src/app
COPY Gemfile Gemfile.lock /src/app/

WORKDIR /src/app
RUN gem install bundler:2.5.3 \
    && bundle config --local deployment 'true' \
    && bundle config --local frozen 'true' \
    && bundle config --local no-cache 'true' \
    && bundle config --local without 'development test cli' \
    && bundle install -j "$(getconf _NPROCESSORS_ONLN)" \
    && find /src/app/vendor/bundle -type f -name '*.c' -delete \
    && find /src/app/vendor/bundle -type f -name '*.h' -delete \
    && find /src/app/vendor/bundle -type f -name '*.o' -delete \
    && find /src/app/vendor/bundle -type f -name '*.gem' -delete

# Install Python Packages
FROM python:${PYTHON_VERSION} AS pip

RUN mkdir -p /src/app
COPY requiremets.txt /src/app/requirements.txt

WORKDIR /src/app
RUN pip install -r /src/app/requirements.txt

# Use "ruby-pycall" as base image
FROM ghcr.io/elct9620/ruby-pycall:${RUBY_VERSION}-py${PYTHON_VERSION}

RUN mkdir -p /src/app
WORKDIR /src/app

# Copy Ruby Gems
COPY --from=gem /usr/local/bundle/config /usr/local/bundle/config
COPY --from=gem /usr/local/bundle /usr/local/bundle
COPY --from=gem /src/app/vendor/bundle /src/app/vendor/bundle

# Copy Python Packages
ARG PYTHON_MINOR_VERSION=3.12
COPY --from=pip /usr/local/lib/python${PYTHON_MINOR_VERSION}/site-packages/ /usr/local/lib/python${PYTHON_MINOR_VERSION}/site-packages

# Copy Source Code
COPY . .

# Run server
CMD ["bundle", "exec", "rails", "server"]
```
