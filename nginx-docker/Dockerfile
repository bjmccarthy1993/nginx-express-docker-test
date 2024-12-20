FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
COPY nginx-certs /etc/nginx/nginx-certs

EXPOSE 80
EXPOSE 443

CMD ["nginx", "-g", "daemon off;"]