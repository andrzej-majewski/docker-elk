# fluentd/Dockerfile
#FROM fluent/fluentd:v0.12-debian
FROM fluent/fluentd:v0.14.25

RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "2.6.1"]
RUN ["gem", "install", "fluent-plugin-detect-exceptions", "--no-rdoc", "--no-ri", "--version", "0.0.8"]
RUN ["gem", "install", "fluent-plugin-concat", "--no-rdoc", "--no-ri", "--version", "2.2.0"]