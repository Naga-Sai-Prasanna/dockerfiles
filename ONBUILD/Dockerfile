# almalinux is exactly same as redhat image,but completely free
FROM almalinux:9
RUN dnf ntsall nginx -y
RUN rm -rf /usr/share/nginx/html/index.html
# this instruction runs when someone use this image
ONBUILD COPY index.html /usr/share/nginx/html/index.html
CMD ["nginx","-g","daemon off;"]