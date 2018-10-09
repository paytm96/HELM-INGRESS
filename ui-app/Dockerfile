#
# BudgetTracker Dockerfile for UI
#
FROM nginx:latest
MAINTAINER Sadhvi Nayak

USER root

# Remove the default Nginx configuration file
RUN rm -v /etc/nginx/nginx.conf

# Copy a configuration file from the current directory
ADD nginx.conf /etc/nginx/

RUN mkdir /etc/nginx/certs
ADD tls.key /etc/nginx/certs/
ADD tls.crt /etc/nginx/certs/
RUN chmod 777 -R /etc/nginx/certs/
ADD web /usr/share/nginx/html/
RUN chmod 777 -R /usr/share/nginx
ADD web /var/www/html/

# Append "daemon off;" to the beginning of the configuration
RUN echo "daemon off;" >> /etc/nginx/nginx.conf

# Expose ports
EXPOSE 443 80

# Set the default command to execute
# when creating a new container
CMD service nginx start
