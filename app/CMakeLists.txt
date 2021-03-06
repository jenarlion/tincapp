# Tinc App, an Android binding and user interface for the tinc mesh VPN daemon
# Copyright (C) 2017-2020 Pacien TRAN-GIRARD
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <https://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.4.1)
include(ExternalProject)

set(CC_OPTIMISATION_FLAGS "-ffunction-sections -fdata-sections")
set(LD_OPTIMISATION_FLAGS "-Wl,--gc-sections")

set(xCONFIG
  "CC=${CMAKE_C_COMPILER} \
    ${CMAKE_C_COMPILE_OPTIONS_EXTERNAL_TOOLCHAIN}${CMAKE_C_COMPILER_EXTERNAL_TOOLCHAIN} \
    ${CMAKE_C_COMPILE_OPTIONS_TARGET}${CMAKE_C_COMPILER_TARGET} \
    ${CMAKE_C_COMPILE_OPTIONS_SYSROOT}${CMAKE_SYSROOT}"
  "LD=${CMAKE_LINKER}"
  "AR=${CMAKE_AR}"
  "RANLIB=${CMAKE_RANLIB}"
  "CFLAGS=${CMAKE_C_FLAGS} ${CC_OPTIMISATION_FLAGS}"
  "LDFLAGS=${CMAKE_SHARED_LINKER_FLAGS} ${LD_OPTIMISATION_FLAGS}"
  "--host=${CMAKE_C_COMPILER_TARGET}"
)

ExternalProject_Add(lzo
  URL               http://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz
  URL_HASH          SHA1=4924676a9bae5db58ef129dc1cebce3baa3c4b5d
  CONFIGURE_COMMAND <SOURCE_DIR>/configure ${xCONFIG} --disable-shared
  BUILD_COMMAND     make -j8
  INSTALL_COMMAND   make install DESTDIR=${CMAKE_CURRENT_BINARY_DIR} &&
                    rm -r <BINARY_DIR>
)

ExternalProject_Add(libressl
  URL               https://ftp.openbsd.org/pub/OpenBSD/LibreSSL/libressl-3.1.4.tar.gz
  URL_HASH          SHA256=414c149c9963983f805a081db5bd3aec146b5f82d529bb63875ac941b25dcbb6
  CONFIGURE_COMMAND <SOURCE_DIR>/configure ${xCONFIG} --disable-shared
  BUILD_COMMAND     make -j8 -C crypto
  INSTALL_COMMAND   make -C crypto install DESTDIR=${CMAKE_CURRENT_BINARY_DIR} &&
                    make -C include install DESTDIR=${CMAKE_CURRENT_BINARY_DIR} &&
                    rm -r <BINARY_DIR>
)

ExternalProject_Add(tinc
  DEPENDS           lzo libressl
  URL               https://github.com/gsliepen/tinc/archive/f5223937e62e1cc5e9b3d322490dd3af8d666750.tar.gz
  URL_HASH          SHA256=3fe923e8fbb1e0192986039e91d6945ffbbe326ee8c2c0a13bacf80e87dad4a9
  # TODO: remove patch once merged in upstream (https://github.com/gsliepen/tinc/pull/251)
  PATCH_COMMAND     patch -p1 < ${PROJECT_SOURCE_DIR}/src/main/c/0001-tincctl-restrict-umask-argument-for-FORTIFY.patch
  CONFIGURE_COMMAND autoreconf -fsi <SOURCE_DIR> &&
                    <SOURCE_DIR>/configure ${xCONFIG}
                      --with-openssl=${CMAKE_CURRENT_BINARY_DIR}/usr/local
                      --with-lzo=${CMAKE_CURRENT_BINARY_DIR}/usr/local
                      --disable-curses
                      --disable-readline
  BUILD_COMMAND     make -j8 -C src
  INSTALL_COMMAND   make -C src install DESTDIR=${CMAKE_CURRENT_BINARY_DIR} &&
                    ${CMAKE_COMMAND} -E copy
                      ${CMAKE_CURRENT_BINARY_DIR}/usr/local/sbin/tinc ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/libtinc.so &&
                    ${CMAKE_COMMAND} -E copy
                      ${CMAKE_CURRENT_BINARY_DIR}/usr/local/sbin/tincd ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/libtincd.so &&
                    rm -r <BINARY_DIR>
)

add_library(main SHARED src/main/c/main.c)
add_dependencies(main tinc)
