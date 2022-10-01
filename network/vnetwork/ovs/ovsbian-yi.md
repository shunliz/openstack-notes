## 直接源码编译安装 {#h2-u76F4u63A5u6E90u7801u7F16u8BD1u5B89u88C5}

    export OVS_VERSION="2.6.1"
    export OVS_DIR="/usr/src/ovs"
    export OVS_INSTALL_DIR="/usr"
    curl -sSl http://openvswitch.org/releases/openvswitch-${OVS_VERSION}.tar.gz | tar -xz && mv openvswitch-${OVS_VERSION} ${OVS_DIR}
    cd ${OVS_DIR}
    ./boot.sh
    # 如果启用DPDK，还需要加上--with-dpdk=/usr/local/share/dpdk/x86_64-native-linuxapp-gcc
    ./configure --prefix=${OVS_INSTALL_DIR} --localstatedir=/var --enable-ssl --with-linux=/lib/modules/$(uname -r)/build
    make -j `nproc`
    make install
    make modules_install

更新内核模块

```
cat > /etc/depmod.d/openvswitch.conf << EOF
override openvswitch * extra
override vport-* * extra
EOF
depmod -a
cp debian/openvswitch-switch.init /etc/init.d/openvswitch-switch
/etc/init.d/openvswitch-switch force-reload-kmod
```

## 编译RPM包 {#h2--rpm-}

```
make rpm-fedora RPMBUILD_OPT="--without check"
```

启用DPDK

```
make rpm-fedora RPMBUILD_OPT="--with dpdk --without check"
```

编译内核模块

```
make rpm-fedora-kmod
```

## 编译deb包 {#h2--deb-}

```
apt-get install build-essential fakeroot
dpkg-checkbuilddeps
# 已经编译过，需要首先clean
# fakeroot debian/rules clean
DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary
```

## 参考文档 {#h2-u53C2u8003u6587u6863}

* [http://docs.openvswitch.org/en/latest/intro/install/](http://docs.openvswitch.org/en/latest/intro/install/)
* [http://docs.openvswitch.org/en/latest/intro/install/debian/](http://docs.openvswitch.org/en/latest/intro/install/debian/)
* [http://docs.openvswitch.org/en/latest/intro/install/dpdk/](http://docs.openvswitch.org/en/latest/intro/install/dpdk/)
* [http://docs.openvswitch.org/en/latest/intro/install/general/\#hot-upgrading](http://docs.openvswitch.org/en/latest/intro/install/general/#hot-upgrading)



