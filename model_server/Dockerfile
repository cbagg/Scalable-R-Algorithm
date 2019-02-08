FROM trestletech/plumber
MAINTAINER Docker User <docker@user.org>

RUN R -e "install.packages('broom')"

COPY plumber.R /opt/plumber.R

ENTRYPOINT ["R", "-e", "pr <- plumber::plumb(commandArgs()[4]); pr$run(host='0.0.0.0', port=8000,swagger=TRUE)"]
CMD ["/opt/plumber.R"]
