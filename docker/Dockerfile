#First stage, start from ubuntu focal
FROM alpine:3 AS base
ENV DEBIAN_FRONTEND noninteractive

RUN apk add --update \
    alpine-sdk \
    fontconfig \
    freetype \
    lua5.3 \
    perl \
    bash \
    python3  \
    libressl-dev \
    py3-pip \
    curl \
    gnupg \
    unzip \
    py3-certifi && \
    rm -rf /var/cache/apk/*

FROM base as base_cert
ENV SSL_CERT_DIR=/usr/local/share/ca-certificates/
ENV DEST_DIR=$SSL_CERT_DIR

WORKDIR certs
RUN openssl s_client -showcerts -verify 5 -connect github.com:443 < /dev/null | awk '/BEGIN/,/END/{ if(/BEGIN/){a++}; out="cert"a".crt"; print >out}' && for cert in *.crt; do mv $cert  "$DEST_DIR/" ; done 
RUN update-ca-certificates

#define environment variables
ENV HTTPS_CA_DIR=$SSL_CERT_DIR
ENV GIT_SSL_CAPATH=$SSL_CERT_DIR/$DEST_CA_FILE
ENV GIT_SSL_CAINFO=$GIT_SSL_CAPATH
ENV REQUESTS_CA_BUNDLE=$SSL_CERT_DIR

#Install opentype fonts
FROM base_cert as font_base

WORKDIR /fonts
ENV DEBIAN_FRONTEND noninteractive

#download fonts
RUN for font in source-sans-pro source-serif-pro source-code-pro; do \
	git_tarball_url="https://www.github.com/adobe-fonts/${font}"$(curl -L "https://github.com/adobe-fonts/${font}/tags" |  \
		     grep -o "/archive/refs/tags/.*\.zip" | grep -v 'variable' | sort -r | head -1 |tr -d '\n'); \
     echo "DOWNLOADING FROM: ${git_tarball_url}"; \
     curl -L --retry 5 "${git_tarball_url}" --output "$font.zip"; \
     unzip "${font}.zip" ; \
    done
RUN mkdir adobe-fonts
RUN find $PWD/ -name "*.ttf" -exec install -m644 {} adobe-fonts/ \; || return 1
RUN rm -rf $PWD/source*

FROM base_cert as pandoc_base

#install python depencies
RUN pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org pip setuptools
RUN pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org --upgrade dicttoxml

#install Pandoc
RUN adduser -s /bin/bash -g "pandoc" -D pandoc

RUN curl --output /tmp/pandoc.tar.gz -L https://github.com/jgm/pandoc/releases/download/2.10.1/pandoc-2.10.1-linux-amd64.tar.gz \
            && tar xvzf /tmp/pandoc.tar.gz --strip-components 1 -C /usr/local/ \
            && update-ca-certificates \
            && rm /tmp/pandoc.tar.gz

# verify pandoc installed correctly
RUN pandoc --version

# install as pandoc
USER pandoc

# setup workdir
WORKDIR /home/pandoc

# Install Lua Filters for pandoc
RUN curl --output /tmp/lua-filters.tar.gz -L https://github.com/pandoc/lua-filters/releases/latest/download/lua-filters.tar.gz  \
    && mkdir /home/pandoc/.pandoc \
    && tar xzf /tmp/lua-filters.tar.gz --strip-components=1 --one-top-level=/home/pandoc/.pandoc/ \
    && rm /tmp/lua-filters.tar.gz

FROM pandoc_base as tinytex_base
#Install tinyTex

# setup path
ENV PATH=/home/pandoc/.TinyTeX/bin/x86_64-linuxmusl/:$PATH
ENV TINY_TEX_VERSION=v0.34
RUN curl -L "https://raw.githubusercontent.com/yihui/tinytex/${TINY_TEX_VERSION}/tools/install-bin-unix.sh" | sed -e 's/wget -qO-/curl -sL/' -e "s#\(sh -s\)#sed 's/wget/curl -LO/' | \1#" > "install-unx.sh" \
	&& chmod +x install-unx.sh
RUN bash -c "./install-unx.sh"

FROM tinytex_base as latex_base

# add tlmgr to path
RUN /home/pandoc/.TinyTeX/bin/*/tlmgr path add

# verify latex version
#RUN latex --version

# verify tlmgr version and add packages needed to build the documents
RUN tlmgr --version 
RUN tlmgr update -self
RUN tlmgr install \
    xecjk \
    ctex \
    fancyhdr \
    ragged2e \
    koma-script \
    setspace \
    colortbl \
    footnotebackref \
    polyglossia \
    pagecolor \
    csquotes \
    caption \
    mdframed \
    needspace \
    titling \
    bookmark \
    newunicodechar \
    adjustbox \
    collectbox \
    listings \
    adjustbox \
    background \
    bidi \
    everypage \
    footmisc \
    fvextra \
    ly1 \
    mweights \
    pagecolor \
    titling \
    ucharcat \
    ulem \
    upquote \
    xurl \
    zref 

from latex_base as work_layer
#copy fonts from font_base
USER root
COPY --from=font_base /fonts/adobe-fonts /usr/share/fonts/opentype/adobe-fonts
RUN fc-cache -f -v "/usr/share/fonts/opentype/adobe-fonts"
USER pandoc

RUN tlmgr info --list
RUN du -a / | sort -n -r | head -n 10

#setup the working dir for running make and creating the files
WORKDIR /data
VOLUME /data

#setup default action
ENTRYPOINT ["/scripts/run.sh"]
