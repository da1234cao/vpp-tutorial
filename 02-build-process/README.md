# vpp build system

## 从源码构建vpp

参考：[Building VPP](https://s3-docs.fd.io/vpp/25.02/developer/build-run-debug/building.html)

```shell
git clone https://gerrit.fd.io/r/vpp
cd vpp

make install-dep
make install-ext-deps
make build
```

## 更多细节

当着手想添加一个 plugin 时，我们会注意到，项目同时包含 make 和 cmake。make 是如何调用 cmake 的呢？

一个 C 项目的构建过程一般是：config -- build -- install 。vpp 的构建过程也遵循这个结构。

在 `build-root/Makefile` 中，我们可以看到 下面这些目标。

```makefile
.PHONY: %-configure
%-configure: %-find-source
	$(configure_check_timestamp)

.PHONY: %-build
%-build: %-configure
	$(build_check_timestamp)

.PHONY: %-install
%-install: %-build
	$(install_check_timestamp)
```

这个具体内容还是得看源码。以 build 过程为例，我这里简单捋下。

```makefile
build_check_timestamp =									\
  @$(BUILD_ENV) ;									\
  comp="$(TIMESTAMP_DIR)/$(BUILD_TIMESTAMP)" ;						\
  conf="$(TIMESTAMP_DIR)/$(CONFIGURE_TIMESTAMP)" ;					\
  dirs="$(call find_source_fn,$(PACKAGE_SOURCE))					\
	$($(PACKAGE)_build_timestamp_depends)						\
	$(if $(is_build_tool),,$(addprefix $(INSTALL_DIR)/,$(PACKAGE_DEPENDENCIES)))" ;	\
  if [[ $${conf} -nt $${comp}								\
        || $(call find_newer_fn, $${comp}, $${dirs}, $?) ]]; then			\
    $(build_package) ;									\ # 这里
    touch $${comp} ;									\
  else											\
    $(call build_msg_fn,Building $(PACKAGE): nothing to do) ;				\
  fi

build_package =							\
  $(call build_msg_fn,Building $* in $(PACKAGE_BUILD_DIR)) ;	\
  mkdir -p $(PACKAGE_BUILD_DIR) ;				\
  cd $(PACKAGE_BUILD_DIR) ;					\
  $(if $($(PACKAGE)_build),					\
       $($(PACKAGE)_build),					\ # 这里
       $(PACKAGE_MAKE))
```

主要调用的是 `$($(PACKAGE)_build)` 。 `$(PACKAGE)_build` 变量中存储着 cmake 的调用命令 。对于 sample-plugin 来说，这个变量定义在 `build-data/packages/sample-plugin.mk` 中。

```makefile
sample-plugin_build = $(CMAKE) --build $(PACKAGE_BUILD_DIR) -- $(MAKE_PARALLEL_FLAGS)
```

## 更多阅读

- [Build System — The Vector Packet Processor v25.02-rc0-121-gaa488dd3f documentation](https://s3-docs.fd.io/vpp/25.02/developer/corearchitecture/buildsystem/index.html)