# BookStack Docker Compose YML File


## 简介
    docker 一键启动 BookStack 容器的 docker-compose.yml 配置


## 镜像环境：
	lscr.io/linuxserver/mariadb
	lscr.io/linuxserver/bookstack


## 导入镜像：
	docker load -i 镜像包


## 启动命令：
	docker-compose up -d


## 安装及生成数据路径：
    ├── bookstack 
		├── fonts                       字体文件目录
        ├── images                      镜像文件目录
        ├── bookstack                       容器挂载目录
            ├── db                        		数据库目录
                └── data                   			数据库数据录
            └── app                        		应用目录
                └── data                   			应用数据目录
        ├── docker-compose.yml          docker-compose.yml
        └── README.md                   README.md


## 隐私信息配置：
	1. 默认账号：admin@admin.com        默认密码：password


## 后续操作：
### 1. 解决 BookStack 导出 PDF 中文乱码 
#### （1）进入 bookstack 容器
	docker exec -it bookstack /bin/bash

#### （2）安装wkhtmltopdf（此操作时间会很长）
	# Copy patches
	RUN mkdir -p /tmp/patches
	COPY conf/* /tmp/patches/

	# Alpine 3.11 and higher versions have libstdc++ and g++ v9+ in their repositories which breaks the build
	RUN echo 'http://dl-cdn.alpinelinux.org/alpine/v3.10/main' >> /etc/apk/repositories

	# Install needed packages
	RUN apk add --no-cache \
	libstdc++=8.3.0-r0 \
	libx11 \
	libxrender \
	libxext \
	libssl3 \
	ca-certificates \
	fontconfig \
	freetype \
	ttf-dejavu \
	ttf-droid \
	ttf-freefont \
	ttf-liberation \
	&& apk add --no-cache --virtual .build-deps \
	g++=8.3.0-r0 \
	git \
	gtk+ \
	gtk+-dev \
	make \
	mesa-dev \
	msttcorefonts-installer \
	openssl-dev \
	patch \
	fontconfig-dev \
	freetype-dev

	# Install microsoft fonts
	&& update-ms-fonts \
	&& fc-cache -f

	# Download source files
	&& git clone --recursive https://github.com/wkhtmltopdf/wkhtmltopdf.git /tmp/wkhtmltopdf \
	&& cd /tmp/wkhtmltopdf \
	&& git checkout tags/0.12.6

	# Apply patches
	&& cd /tmp/wkhtmltopdf/qt \
	&& patch -p1 -i /tmp/patches/qt-musl.patch \
	&& patch -p1 -i /tmp/patches/qt-musl-iconv-no-bom.patch \
	&& patch -p1 -i /tmp/patches/qt-recursive-global-mutex.patch \
	&& patch -p1 -i /tmp/patches/qt-gcc6.patch

	# Modify qmake config
	&& sed -i "s|-O2|$CXXFLAGS|" mkspecs/common/g++.conf \
	&& sed -i "/^QMAKE_RPATH/s| -Wl,-rpath,||g" mkspecs/common/g++.conf \
	&& sed -i "/^QMAKE_LFLAGS\s/s|+=|+= $LDFLAGS|g" mkspecs/common/g++.conf

	# Prepare optimal build settings
	&& NB_CORES=$(grep -c '^processor' /proc/cpuinfo)

	# Install qt
	&& ./configure -confirm-license -opensource \
	-prefix /usr \
	-datadir /usr/share/qt \
	-sysconfdir /etc \
	-plugindir /usr/lib/qt/plugins \
	-importdir /usr/lib/qt/imports \
	-silent \
	-release \
	-static \
	-webkit \
	-script \
	-svg \
	-exceptions \
	-xmlpatterns \
	-openssl-linked \
	-no-fast \
	-no-largefile \
	-no-accessibility \
	-no-stl \
	-no-sql-ibase \
	-no-sql-mysql \
	-no-sql-odbc \
	-no-sql-psql \
	-no-sql-sqlite \
	-no-sql-sqlite2 \
	-no-qt3support \
	-no-opengl \
	-no-openvg \
	-no-system-proxies \
	-no-multimedia \
	-no-audio-backend \
	-no-phonon \
	-no-phonon-backend \
	-no-javascript-jit \
	-no-scripttools \
	-no-declarative \
	-no-declarative-debug \
	-no-mmx \
	-no-3dnow \
	-no-sse \
	-no-sse2 \
	-no-sse3 \
	-no-ssse3 \
	-no-sse4.1 \
	-no-sse4.2 \
	-no-avx \
	-no-neon \
	-no-rpath \
	-no-nis \
	-no-cups \
	-no-pch \
	-no-dbus \
	-no-separate-debug-info \
	-no-gtkstyle \
	-no-nas-sound \
	-no-opengl \
	-no-openvg \
	-no-sm \
	-no-xshape \
	-no-xvideo \
	-no-xsync \
	-no-xinerama \
	-no-xcursor \
	-no-xfixes \
	-no-xrandr \
	-no-mitshm \
	-no-xinput \
	-no-xkb \
	-no-glib \
	-no-icu \
	-nomake demos \
	-nomake docs \
	-nomake examples \
	-nomake tools \
	-nomake tests \
	-nomake translations \
	-graphicssystem raster \
	-qt-zlib \
	-qt-libpng \
	-qt-libmng \
	-qt-libtiff \
	-qt-libjpeg \
	-optimized-qmake \
	-iconv \
	-xrender \
	-fontconfig \
	-D ENABLE_VIDEO=0 \
	&& make --jobs $(($NB_CORES*2)) --silent \
	&& make install

	Install wkhtmltopdf
	&& cd /tmp/wkhtmltopdf \
	&& qmake \
	&& make --jobs $(($NB_CORES*2)) --silent \
	&& make install \
	&& make clean \
	&& make distclean

#### （3）配置 bookstac 使用 kwkhtmltopdf，编辑 /app/www/.env，新增配置如下：
	WKHTMLTOPDF=/bin/wkhtmltopdf
	ALLOW_UNTRUSTED_SERVER_FETCHING=true

#### （4）系统安装字体
	1. 放入字体：simsun.ttc 放入 /usr/share/fonts/chinese/
	2. 刷新字体：fc-cache -fv

#### （5）重启 bookstack 容器
	docker restart bookstack


