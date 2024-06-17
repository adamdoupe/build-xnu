#
# Apple XNU makefile
# Built-on Jeremy C. Andrus' work: https://kernelshaman.blogspot.com/2021/02/building-xnu-for-macos-112-intel-apple.html
# 

macos_version ?= 11.2

PKGS = dtrace AvailabilityVersions libdispatch xnu Libsystem
dtrace_version=370.40.1
dtrace_src=dtrace-dtrace-370.40.1
AvailabilityVersions_version=70
AvailabilityVersions_src=AvailabilityVersions-AvailabilityVersions-70
xnu_version=7195.81.3
xnu_src=xnu-xnu-7195.81.3
libdispatch_version=1271.40.12
libdispatch_src=libdispatch-libdispatch-1271.40.12

# there are tools that need to be compiled from source that are required to build, let's put them here
OUR_TOOL_PATH=$(shell pwd)/bin
OUR_LIB_PATH=$(shell pwd)/include

export PATH := $(OUR_TOOL_PATH):${PATH}
export CPATH := $(OUR_LIB_PATH)/usr/local/include:${CPATH}

default: xnudeps
xnudeps: check_tools
xnu: check_tools

make_tool_path:
	mkdir -p $(OUR_TOOL_PATH)

make_lib_path:
	mkdir -p $(OUR_LIB_PATH)

APPLEOPENSOURCE = https://web.archive.org/web/20221007131956/https://opensource.apple.com
TARBALL_URL = https://opensource.apple.com/tarballs
RELEASE_URL = $(APPLEOPENSOURCE)/release/macos-$(subst .,,$(macos_version)).html

TOOLCHAIN_PATH=$(shell cd `xcrun -sdk macosx -show-sdk-platform-path`/../../Toolchains/XcodeDefault.xctoolchain && pwd)
SDKPATH=$(shell xcrun -sdk macosx -show-sdk-path)
XCODEPATH=$(shell xcrun -sdk macosx -show-sdk-path | sed 's,\(.*/Xcode[^/]*\.app\)/.*,\1,')

dtrace: dtrace-11.x

.SECONDEXPANSION:
dtrace-11.x: $$(dtrace_src)/.src

dtrace-11.x: make_tool_path
	@echo "[INFO] Building ctf tools from dtrace..."
	@$(HELP) cd "$(SRCDIR)" && $(HELP) xcodebuild install -sdk macosx -target ctfconvert \
		-target ctfdump -target ctfmerge ARCHS='x86_64 arm64' VALID_ARCHS='x86_64 arm64' DSTROOT="$(SRCDIR)/dst" CODE_SIGNING_ALLOWED=NO
	cp $(SRCDIR)/dst/$(TOOLCHAIN_PATH)/usr/local/bin/* "$(OUR_TOOL_PATH)"

AvailabilityVersions: make_tool_path
	@echo "[INFO] Installing AvailabilityVersions content..."
	@$(HELP) mkdir -p "$(SRCDIR)/dst"
	@$(HELP) rm -rf "$(SRCDIR)/dst/"*
	@$(HELP) make -C "$(SRCDIR)" install SRCROOT="$(SRCDIR)" DSTROOT="$(SRCDIR)/dst"
	cp $(SRCDIR)/dst/usr/local/libexec/* "$(OUR_TOOL_PATH)"

libplatform: INTPATH = usr/local/include/os/internal
libplatform:
	@echo "[INFO] Installing libplatform headers to SDK..."
	$(HELP) sudo mkdir -p "$(SDKPATH)/$(INTPATH)"
	$(HELP) sudo ditto "$(SRCDIR)/private/os/internal" "$(SDKPATH)/$(INTPATH)"

libdispatch: libdispatch-11.x
libdispatch: ARCHS=x86_64 arm64e

.SECONDEXPANSION:
libdispatch-11.x: $$(libdispatch_src)/.src

libdispatch-11.x: xnu-installhdrs
	@echo "[INFO] Building libfirehose_kernel.a from libdispatch..."
	# next we need to install the firehose headers for the kernel
	@$(HELP) cd $(SRCDIR) && CPATH=$(OUR_LIB_PATH)/System/Library/Frameworks/Kernel.framework/PrivateHeaders:$(OUR_LIB_PATH)/System/Library/Frameworks/Kernel.framework/Headers:$(OUR_LIB_PATH)/usr/local/include/firehose:$(CPATH) $(HELP) xcodebuild install -sdk macosx -target libfirehose_kernel PRODUCT_NAME=firehose_kernel ARCHS='$(ARCHS)' VALID_ARCHS='$(ARCHS)' SRCROOT="$(SRCDIR)" OBJROOT="$(SRCDIR)/obj" SYMROOT="$(SRCDIR)/sym" DSTROOT="$(SRCDIR)/dst" CODE_SIGNING_ALLOWED=NO
	$(HELP) ditto "$(SRCDIR)/dst/" "$(OUR_LIB_PATH)"

xnu-installhdrs: ARCHS=X86_64 ARM64

xnu-installhdrs: dtrace AvailabilityVersions make_lib_path xnu-fixups
	@echo "[INFO] Installing XNU headers into the SDK..."	
	@$(HELP) cd "$(xnu_path)" && $(HELP) make XNU_LOGCOLORS=y SDKROOT=macosx ARCH_CONFIGS="$(ARCHS)" installhdrs
	ditto "$(xnu_path)/BUILD/dst/" "$(OUR_LIB_PATH)"

.SECONDEXPANSION:
xnu-installhdrs: $$(xnu_src)/.src

xnu: dtrace AvailabilityVersions libdispatch xnu-fixups
	cd $(xnu_src) && CPATH=$(OUR_LIB_PATH)/usr/local/include/kernel/ LIBRARY_PATH=$(OUR_LIB_PATH)/usr/local/lib/kernel/ make SDKROOT=macosx ARCH_CONFIGS="X86_64" KERNEL_CONFIGS="RELEASE DEVELOPMENT" PYTHON=python2
.PHONY: xnu-fixups
xnu-fixups: $$(xnu_src)/.src
	patch -N -r- -p1 < xnu.patch || true

Libsystem: ARCHS = x86_64 arm64
libsyscall: ARCHS = x86_64 arm64

Libsystem:
	@$(HELP) mkdir -p $(SRCDIR)/dst $(SRCDIR)/obj $(SRCDIR)/sym
	@$(HELP) cd $(SRCDIR) && $(HELP) xcodebuild installhdrs -sdk macosx ARCHS='$(ARCHS)' VALID_ARCHS='$(ARCHS)' SRCROOT="$(SRCDIR)" OBJROOT="$(SRCDIR)/obj" DSTROOT="$(SRCDIR)/dst"
	$(HELP) sudo ditto "$(SRCDIR)/dst" "$(SDKPATH)"

libsyscall: BUILDDIR=$(xnu_path)/BUILD.libsyscall
libsyscall: dtrace AvailabilityVersions libdispatch Libsystem
	# install XNU and Libsyscall headers
	@$(HELP) mkdir -p "$(xnu_path)/BUILD.hdrs/dst" "$(xnu_path)/BUILD.hdrs/obj" "$(xnu_path)/BUILD.hdrs/sym"
	@$(HELP) cd "$(xnu_path)" && $(HELP) make installhdrs XNU_LOGCOLORS=y SDKROOT=macosx ARCH_CONFIGS=X86_64 SRCROOT="$(xnu_path)" OBJROOT="$(xnu_path)/BUILD.hdrs/obj" SYMROOT="$(xnu_path)/BUILD.hdrs/sym" DSTROOT="$(xnu_path)/BUILD.hdrs/dst"
	@$(HELP) cd "$(xnu_path)" && $(HELP) make installhdrs XNU_LOGCOLORS=y RC_ProjectName=Libsyscall SDKROOT=macosx RC_ARCHS='$(ARCHS)' VALID_ARCHS='$(ARCHS)' OBJROOT="$(xnu_path)/BUILD.hdrs/obj" SYMROOT="$(xnu_path)/BUILD.hdrs/sym" DSTROOT="$(xnu_path)/BUILD.hdrs/dst"
	$(HELP) sudo ditto "$(xnu_path)/BUILD.hdrs/dst" "$(SDKPATH)"
	# Build Libsyscall
	@$(HELP) mkdir -p "$(BUILDDIR)/dst" "$(BUILDDIR)/obj" "$(BUILDDIR)/sym"
	@$(HELP) cd "$(xnu_path)" && $(HELP) make install XNU_LOGCOLORS=y RC_ProjectName=Libsyscall RC_ARCHS='x86_64 i386' SDKROOT=macosx OBJROOT="$(BUILDDIR)/obj" SYMROOT="$(BUILDDIR)/sym" DSTROOT="$(BUILDDIR)/dst"

help-libsyscall: HELP=echo CMD: 
help-libsyscall: libsyscall

.SECONDEXPANSION:
libsyscall: $$(xnu_src)/.src

libsyscall_install: libsyscall
	$(HELP) sudo ditto "$(xnu_src)/BUILD.libsyscall/dst/usr/lib/system" "/usr/lib/system"
	$(HELP) sudo update_dyld_shared_cache
	@echo
	@echo "A new libsyscall has been installed in: /usr/lib/system"
	@echo "You must reboot for this change to take effect."
	@echo


help-libsyscall_install: HELP=echo CMD: 
help-libsyscall_install: libsyscall

download:
	@echo "All source downloaded"

define download_tarball
$(1)_version ?= $$(shell curl -s $$(RELEASE_URL) | grep "$(1)" | grep tarballs/ | cut -d '-' -f 2 | sed 's/.tar.gz.*//')
$(1)_tarball := $(1)-$$($(1)_version).tar.gz
$(1)_tarball_url := $$(TARBALL_URL)/$(1)/$(1)-$$($(1)_version).tar.gz
$(1)_src ?= $(1)-$$($(1)_version)
$(1)_path ?= $$(shell pwd)/$$($(1)_src)
$(1)_ver_err := Couldn't determine version of $(1) ($$($(1)_version)) from $$(RELEASE_URL). Are you connected to the Internet?
#$(1)-$$($(1)_version).tar.gz:
$$($(1)_tarball):
	@echo "Downloading $(1) v$$($(1)_version)..."
#	@if [ -z "$$($(1)_version)" ]; then $(MAKE) -f $(lastword $(MAKEFILE_LIST)) ERROR="$$($(1)_ver_err)" err; fi
	@$$(HELP) curl -L -O $$($(1)_tarball_url)
$$($(1)_src)/.src: $$($(1)_tarball)
	@echo "Unpacking $(1)..."
	@$$(HELP) tar zxf $$<
	@$$(HELP) touch $$@
$(1): $$($(1)_src)/.src
$(1): SRCDIR=$$($(1)_path)
.PHONY: clean-$(1) $$($(1)_vers)
clean-$(1):
	rm -rf $$($(1)_src)
	rm -f $$($(1)_tarball)
help-$(1): HELP=echo CMD: 
help-$(1): $(1)
	echo 
download: $$($(1)_src)/.src
download-$(1): $$($(1)_tarball)
ver-$(1):
	@if [ -z "$$($(1)_version)" ]; then $(MAKE) -f $(lastword $(MAKEFILE_LIST)) ERROR="$$($(1)_ver_err)" err; fi
	@echo "$(1)-$$($(1)_version) : $$($(1)_tarball_url)"
endef

$(foreach pkg,$(PKGS),$(eval $(call download_tarball,$(pkg))))

getvers: $(foreach pkg,$(PKGS),ver-$(pkg))
	@echo "macOS: $(macos_version)"

err:
	$(error $(ERROR))

clean: $(foreach pkg,$(PKGS),clean-$(pkg))

help: check_tools
	@echo "Available targets:"
	@echo "    xnudeps"
	@echo "    download"
	@echo "    clean"
	@echo "    $(PKGS) libsyscall libsyscall_install"
	@echo "    $(PKGS:%=help-%) help-libsyscall help-libsyscall_install"
	@echo
	@echo "default: xnudeps"
	@echo
	@echo "Use the macos_version variable to set the version of "
	@echo "macOS for which you would like to compile, e.g."
	@echo
	@echo "default: macos_version=$(macos_version)"
	@echo
	@echo "TOOLCHAIN_PATH: $(TOOLCHAIN_PATH)"
	@echo "SDKPATH: $(SDKPATH)"
	@echo "XCODEPATH: $(XCODEPATH)"

.PHONY: err getvers help clean check_tools

