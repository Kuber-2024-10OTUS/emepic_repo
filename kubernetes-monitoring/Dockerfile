FROM nginx:latest

# Установка необходимых инструментов
RUN apt-get update && apt-get install -y wget

# Скачивание и установка nginx-prometheus-exporter
RUN wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v0.10.0/nginx-prometheus-exporter_0.10.0_linux_amd64.tar.gz
RUN tar -xzf nginx-prometheus-exporter_0.10.0_linux_amd64.tar.gz
RUN mv nginx-prometheus-exporter /usr/local/bin/

# Копирование конфигурации Nginx
COPY nginx.conf /etc/nginx/nginx.conf

# Запуск Nginx и экспортера
CMD nginx -g "daemon off;" & nginx-prometheus-exporter -nginx.scrape-uri http://localhost/status