# build in root dir using:
#
#     docker build -t roundup-app --rm -f scripts/Dockerfile .
#
# run using:
#
#    docker run --rm -v /.../issue.tracker:/usr/src/app/tracker \
#           -p 9017:8080 roundup-app:latest


# Global vars for all build stages

# application directory
ARG appdir=/usr/src/app

# support roundup install from 'local' directory or from 'pypi'
ARG source=pypi

FROM python:3-alpine as build

# Inherit global values https://github.com/moby/moby/issues/37345
ARG appdir

WORKDIR $appdir

# Add packages needed to compile mysql, pgsql and other python modules.
# Can't use apk to add them as that installs a 3.9 python version.
#        g++ installs cc1plus needed by pip install
RUN apk add \
    g++ \
    gcc \
    gpgme-dev \
    linux-headers \
    musl-dev \
    mysql-dev \
    postgresql-dev \
    swig

# add requirements for pip here, e.g. Whoosh, gpg, zstd or other
#   modules not installed in the base library.
# ignore warnings from pip to use virtualenv
COPY ./scripts/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# copy the elements of the release directory to the docker image
COPY setup.py install/
COPY doc install/doc/
COPY frontends install/frontends/
COPY locale install/locale/
COPY roundup install/roundup/
COPY share install/share/

# verify source has one of two valid values then
# install in python3 standard directories from local copy
# or install in python3 standard directories from pypi using pip
# import from global/command line
ARG source
RUN set -xv && if [ "$source" = "local" ] || [ "$source" = "pypi" ]; then :; \
           else echo "invalid value for source: $source"; \
                echo "must be local or pypi"; exit 1; fi; \
    if [ "$source" = "local" ]; then cd install && ./setup.py install; fi; \
    if [ "$source" = "pypi" ]; then pip install roundup; \
        cp -ril /usr/local/lib/python3.10/site-packages/usr/local/share/* \
           /usr/local/share; fi

# build a new smaller docker image for execution. Build image above
# is 1G in size.
FROM python:3-alpine

# import from global
ARG appdir

WORKDIR $appdir

# add libraries needed to run gpg/mysql/pgsql/brotli
RUN apk add \
     gpgme \
     mariadb-connector-c \
     libpq \
     libstdc++

ARG source
LABEL "org.roundup-tracker.vendor"="Roundup Issue Tracker Team" \
      "org.roundup-tracker.description"="Roundup Issue Tracker using sqlite" \
      "version"="2.1.0 $source"

# pull over built assets
COPY --from=build /usr/local/lib/python3.10/site-packages /usr/local/lib/python3.10/site-packages/
COPY --from=build /usr/local/bin/roundup* /usr/local/bin/
COPY --from=build /usr/local/share /usr/local/share/

# map port 8080 to your local port
EXPOSE 8080/tcp

# mount a trackerdir on tracker location
RUN mkdir tracker
VOLUME $appdir/tracker

# do not run roundup as root
RUN adduser -D -h /usr/src/app roundup
USER roundup

# run the server, disable output buffering so we can see logs.
ENV PYTHONUNBUFFERED=1
ENTRYPOINT [ "roundup-server", "-n", "0.0.0.0" ]

# allow the invoker to override cmd with multiple trackers
# in each subdirectory under $appdir/tracker. E.G.
# docker run .... \
#   issues=tracker/issues foo=tracker/foo
#
# note using "issue=$appdir/tracker" results in error:
#
#  No valid configuration files found in directory /usr/src/app/$appdir/tracker
#
# so $appdir not expanded and $PWD prefixed onto the (relative path)
#   $appdir/tracker. Hence use relative path for spec.
CMD [  "issues=tracker" ]
