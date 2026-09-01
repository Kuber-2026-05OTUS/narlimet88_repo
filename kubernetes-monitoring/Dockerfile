FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/nginx.conf

RUN echo '<html><body><h1>OTUS Kubernetes Monitoring</h1></body></html>' \
    > /usr/share/nginx/html/index.html

EXPOSE 80
