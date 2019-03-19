FROM python:3.7-alpine

RUN apk --no-cache add g++ linux-headers \
    && pip install locustio

EXPOSE 8089 5557 5558
ADD ./locust /locust
RUN chmod -R 755 /locust
ENTRYPOINT ["/locust/run.sh"] 
