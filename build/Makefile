# -*- mode: Makefile; -*-

#-------------------------------------------------------------------------------
# 2015-08-14
#-------------------------------------------------------------------------------

BASE_DIR := $(subst /build,,$(PWD))
OMI_ROOT := $(shell cd ../../omi/Unix; pwd -P)
PAL_DIR := $(shell cd ../../pal; pwd -P)
SCX_DIR := $(shell cd ../../opsmgr; pwd -P)
DSC_DIR := $(shell cd ../../dsc; pwd -P)

PF_POSIX := 1
include $(BASE_DIR)/build/config.mak
include $(BASE_DIR)/build/Makefile.version
include $(PAL_DIR)/build/Makefile.pal

ifndef ENABLE_DEBUG
$(error "ENABLE_DEBUG is not set.  Please re-run configure")
endif

SCX_TARGET_DIR := $(SCX_DIR)/target/$(BUILD_CONFIGURATION)
DSC_TARGET_DIR := $(DSC_DIR)/release

RUBY_DIR := $(BASE_DIR)/source/ext/ruby
FLUENTD_DIR := $(BASE_DIR)/source/ext/fluentd
PLUGINS_DIR := $(BASE_DIR)/source/code/plugins

INTERMEDIATE_DIR=$(BASE_DIR)/intermediate/$(BUILD_CONFIGURATION)
TARGET_DIR := $(BASE_DIR)/target/$(BUILD_CONFIGURATION)

ifeq ($(ULINUX),1)
# Doesn't matter what version of SSL/Ruby we use to compile or link the OMI plugin
RUBY_DEST_DIR := $(INTERMEDIATE_DIR)/098/ruby
OMI_LIBRARY_DIR := $(OMI_ROOT)/output_openssl_0.9.8/lib
else
RUBY_DEST_DIR := $(INTERMEDIATE_DIR)/ruby
OMI_LIBRARY_DIR := $(OMI_ROOT)/output/lib
endif

# Version of Ruby for test purposes only
RUBY_TESTING_DIR := $(shell echo "$(RUBY_CONFIGURE_QUALS_TESTINS)" | cut -d= -f2)
RUBY_TESTING_EXE := $(RUBY_TESTING_DIR)/bin/ruby

# Compiler flags

ifeq ($(ENABLE_DEBUG),1)
DEBUG_FLAGS := -g
endif

# Need to use RUBY_COMPILE_FLAGS when compiling code that uses C Ruby interfaces
# (Note that "-Wshadow -Wredundant-decls" don't work with Ruby v2.2.0)
RUBY_COMPILE_FLAGS := $(DEBUG_FLAGS) -D_REENTRANT -fstack-protector-all -Wall -fno-nonansi-builtins -Woverloaded-virtual -Wformat -Wformat-security -Wcast-align -Wswitch-enum  -Wwrite-strings  -Werror -Wcast-qual -fPIC # -Wshadow -Wredundant-decls
PLUGIN_LINK_LIBRARIES := -L$(OMI_LIBRARY_DIR) -lrt -pthread -lmiapi -lprotocol -lsock -lbase -lpal -lwsman
SHARED_FLAGS := -shared

# Support for installbuilder

STAGING_DIR := $(INTERMEDIATE_DIR)/staging
INSTALLER_TMPDIR := $(INTERMEDIATE_DIR)/installer_tmp

INSTALLER_DATAFILES := base_omsagent.data linux.data ruby.data
INSTALLER_DATAFILES_RPM := $(INSTALLER_DATAFILES) linux_rpm.data
INSTALLER_DATAFILES_DPKG := $(INSTALLER_DATAFILES) linux_dpkg.data

ifeq ($(PACKAGE_SUFFIX),rpm)
INSTALLER_DATAFILES_CURRENT := $(INSTALLER_DATAFILES_RPM)
else
INSTALLER_DATAFILES_CURRENT := $(INSTALLER_DATAFILES_DPKG)
endif

ifeq ("$(wildcard /usr/bin/dpkg-deb)","")
DPKG_LOCATION="--DPKG_LOCATION=$(PAL_DIR)/installer/InstallBuilder/tools/bin/dpkg-deb-$(PF_ARCH)"
else
DPKG_LOCATION=
endif

# Other packaging support

ifeq ($(ULINUX),1)
  ifeq ($(PF_ARCH),x86)
    OSS_KITS := $(shell cd $(SCX_DIR)/installer/oss-kits; ls *-oss-test.sh *.i686.sh)
    OMS_OSS_KITS := $(shell cd $(BASE_DIR)/installer/oss-kits; ls *-oss-test.sh *.i686.sh)
  else ifeq ($(PF_ARCH),x64)
    OSS_KITS := $(shell cd $(SCX_DIR)/installer/oss-kits; ls *-oss-test.sh *.x86_64.sh)
    OMS_OSS_KITS := $(shell cd $(BASE_DIR)/installer/oss-kits; ls *-oss-test.sh *.x86_64.sh)
  endif
endif

# Plugins
IN_PLUGINS_LIB := $(INTERMEDIATE_DIR)/Libomi.so

OMI_INCLUDES := -I$(OMI_ROOT) -I$(OMI_ROOT)/common -I$(OMI_ROOT)/common/linux -I$(OMI_ROOT)/base -I$(OMI_ROOT)/output/include
PAL_INCLUDES := -I$(PAL_DIR)/source/code/include/util
RUBY_INCLUDES := -I$(RUBY_DEST_DIR)/include/ruby-2.2.0/$(RUBY_ARCM) -I$(RUBY_DEST_DIR)/include/ruby-2.2.0

ifeq ($(ULINUX),1)
  OUTPUT_PACKAGE_PREFIX=omsagent-$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH)-$(OMS_BUILDVERSION_BUILDNR).universal.$(PF_ARCH)
else
  PF_DISTRO_LC := $(shell echo $(PF_DISTRO) | tr A-Z a-z)
  OUTPUT_PACKAGE_PREFIX=omsagent-$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH)-$(OMS_BUILDVERSION_BUILDNR).$(PF_DISTRO_LC).$(PF_MAJOR).$(PF_ARCH)
endif

#--------------------------------------------------------------------------------
# Build targets
.PHONY: all clean clean-ruby distclean clean-status plugins-status kit
.PHONY: tests test omstest unittest systemtest systemtestrb systemtestsh

all : $(SCX_TARGET_DIR) $(DSC_TARGET_DIR) $(RUBY_TESTING_DIR) $(RUBY_DEST_DIR) plugins-status $(IN_PLUGINS_LIB) kit

clean : clean-status
	$(RMDIR) $(INTERMEDIATE_DIR)/source/code

	# On Debian, installer staging directory has root permissions, so sudo elevate
	sudo $(RMDIR) $(STAGING_DIR) $(INSTALLER_TMPDIR) $(BASE_DIR)/target 

	$(RM) $(IN_PLUGINS_LIB)
	-find $(BASE_DIR) -name \*~ -exec rm {} \;

clean-ruby : clean
	sudo $(RMDIR) $(BASE_DIR)/intermediate
	sudo $(RMDIR) $(RUBY_DIR)/.ext
	#
	# Warning: This step will clean out checked out files from Ruby directory
	#
	@echo "Cleaning RUBY source directory ..."
	find $(RUBY_DIR) -type f -perm -u+w -exec rm -f {} \;
	#
	# Warning: This step will clean out checked out files from fluentd directory
	#
	@echo "Cleaning fluentd source directory ..."
	find $(FLUENTD_DIR) -type f -perm -u+w -print -exec rm {} \;

clean-dsc : clean
	-make -C $(DSC_DIR) clean
	-$(RMDIR) $(DSC_DIR)/config.mak $(DSC_DIR)/intermediate $(DSC_DIR)/output $(DSC_DIR)/release
	-$(RM) $(DSC_DIR)/omi-1.0.8

clean-scx : clean
	-make -C $(SCX_DIR)/build distclean
	@$(ECHO) "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!"
	@$(ECHO) "!!!!! PLEASE RE-RUN THE ./configure SCRIPT !!!!!"
	@$(ECHO) "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!"

distclean : clean clean-ruby clean-dsc clean-scx
	-$(RMDIR) $(OMI_ROOT)/output*

	$(RM) $(BASE_DIR)/build/config.mak
	-$(RM) $(PAL_DIR)/build/config.mak

clean-status :
	@$(ECHO) "========================= Performing make clean"

#--------------------------------------------------------------------------------
# Build SCX project (special mode to only build the provider itself if ulinux)

$(SCX_TARGET_DIR) : 
	$(MAKE) -C $(SCX_DIR)/build all

#--------------------------------------------------------------------------------
# Build DSC project (configured for the OMS project)
#
# Note: This expects that SCX (and thus OMI) is already built! Take care of your
# ordering on the 'all' target!

$(DSC_TARGET_DIR) :
ifeq ($(ULINUX),1)
	@$(ECHO) "========================= Performing Building DSC Project"
	cd $(DSC_DIR); ./configure --oms
	ln -fs $(OMI_ROOT) $(DSC_DIR)/omi-1.0.8
	cd $(OMI_ROOT); $(RMDIR) output; ln -s output_openssl_0.9.8 output
	$(MAKE) -C $(DSC_DIR) dsc098
	cd $(OMI_ROOT); $(RMDIR) output; ln -s output_openssl_1.0.0 output
	$(MAKE) -C $(DSC_DIR) dsc100
	cd $(OMI_ROOT); $(RMDIR) output
endif

#--------------------------------------------------------------------------------
# Build the version of Ruby that we test with

$(RUBY_TESTING_DIR) :
	#
	# Warning: This step will clean out checked out files from both Ruby and fluentd directories
	#
	@$(ECHO) "========================= Performing Building Ruby for testing"
	sudo $(RMDIR) /usr/local/ruby-2.2.0*
	chmod +x $(BASE_DIR)/build/buildRuby.sh; $(BASE_DIR)/build/buildRuby.sh test

# Build the version of Ruby that we distribute

$(RUBY_DEST_DIR) :
	#
	# Warning: This step will clean out checked out files from both Ruby and fluentd directories
	#
ifeq ($(ULINUX),1)
	@$(ECHO) "========================= Performing Building Ruby for SSL v0.9.8 ..."
	chmod +x $(BASE_DIR)/build/buildRuby.sh; $(BASE_DIR)/build/buildRuby.sh 098
	@$(ECHO) "========================= Performing Building Ruby for SSL v1.0.0 ..."
	chmod +x $(BASE_DIR)/build/buildRuby.sh; $(BASE_DIR)/build/buildRuby.sh 100
else
	@$(ECHO) "========================= Performing Building Ruby"
	chmod +x $(BASE_DIR)/build/buildRuby.sh; $(BASE_DIR)/build/buildRuby.sh
endif

#================================================================================
# Internal functions
#================================================================================

# Convert a list of src files with absolute paths under BASE_DIR to corresponding
# object files under intermediate directory
# src_to_obj(list_of_cppfiles)
src_to_obj = $(patsubst $(BASE_DIR)%, $(INTERMEDIATE_DIR)%, $(patsubst %.c, %.o, $(patsubst %.cpp, %.o, $(1))))

# No default rules, please
.SUFFIX:

# Rule for compiling cpp files in source tree, ouptut in mirrored intermediate dir
$(INTERMEDIATE_DIR)/%.o : $(BASE_DIR)/%.cpp
	$(MKPATH) $(@D)
	$(CXX) -c $(CXXFLAGS) $(INCLUDES) -I$(<D) -o $@ $<

$(INTERMEDIATE_DIR)/%.o : $(BASE_DIR)/%.c
	$(MKPATH) $(@D)
	$(CXX) -c $(CFLAGS) $(INCLUDES) -I$(<D) -o $@ $<

#--------------------------------------------------------------------------------
# Build the plugins that we require for our agent

STATIC_PLUGINS_SRCFILES = $(PLUGINS_DIR)/omi_interface.cpp
STATIC_PLUGINS_OBJFILES = $(call src_to_obj,$(STATIC_PLUGINS_SRCFILES))

$(IN_PLUGINS_LIB) : CXXFLAGS = $(RUBY_COMPILE_FLAGS)
$(IN_PLUGINS_LIB) : INCLUDES = $(RUBY_INCLUDES) $(OMI_INCLUDES) $(PAL_INCLUDES)
$(IN_PLUGINS_LIB) : $(STATIC_PLUGINS_OBJFILES)
	$(MKPATH) $(INTERMEDIATE_DIR)
	$(MKPATH) $(TARGET_DIR)
	g++ $(SHARED_FLAGS) $(RUBY_INCLUDES) $(OMI_INCLUDES) $(PAL_INCLUDES) -o $@ $(STATIC_PLUGINS_OBJFILES) $(PLUGIN_LINK_LIBRARIES)

plugins-status :
	@echo "========================= Performing Building input plugins"

#--------------------------------------------------------------------------------
# Build the distribution kit
#
# Build the packages via installbuilder
#
# While the "formal build" only builds ULINUX, we may build something else for DEV purposes.
# Assume we ALWAYS build RPM, but only build DPKG if --enable-ulinux is speified in configure.

ifeq ($(ULINUX),1)

kit : $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).sh

$(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).sh : $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).tar
	@echo "========================= Performing Building shell bundle"
	chmod +x ../installer/bundle/create_bundle.sh
	../installer/bundle/create_bundle.sh $(TARGET_DIR) $(INTERMEDIATE_DIR) $(OUTPUT_PACKAGE_PREFIX).tar

$(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).tar : $(RUBY_DEST_DIR) $(IN_PLUGINS_LIB) \
	$(TARGET_DIR)/098/$(OUTPUT_PACKAGE_PREFIX).rpm $(TARGET_DIR)/098/$(OUTPUT_PACKAGE_PREFIX).deb \
	$(TARGET_DIR)/100/$(OUTPUT_PACKAGE_PREFIX).rpm $(TARGET_DIR)/100/$(OUTPUT_PACKAGE_PREFIX).deb

	@echo "========================= Performing Building .tar file"

	# Gather the OMI bits that we need and rename them in a non-SCX manner (easier to deal with)
	# Note: We take care to only copy the latest version of OMI if there are multiple versions
	$(RM) $(TARGET_DIR)/098/omi-*.{rpm,deb} $(TARGET_DIR)/100/omi-*.{rpm,deb}

	cd $(OMI_ROOT)/output_openssl_0.9.8/release; $(COPY) `ls omi-*.rpm | sort | tail -1` $(TARGET_DIR)/098
	cd $(OMI_ROOT)/output_openssl_0.9.8/release; $(COPY) `ls omi-*.deb | sort | tail -1` $(TARGET_DIR)/098
	cd $(TARGET_DIR)/098; SOURCE=`ls omi-*.rpm` ; DEST=`echo $$SOURCE | sed 's/r.1././'` ; $(MV) $$SOURCE $$DEST
	cd $(TARGET_DIR)/098; SOURCE=`ls omi-*.deb` ; DEST=`echo $$SOURCE | sed 's/d.1././'` ; $(MV) $$SOURCE $$DEST

	cd $(OMI_ROOT)/output_openssl_1.0.0/release; $(COPY) `ls omi-*.rpm | sort | tail -1` $(TARGET_DIR)/100
	cd $(OMI_ROOT)/output_openssl_1.0.0/release; $(COPY) `ls omi-*.deb | sort | tail -1` $(TARGET_DIR)/100
	cd $(TARGET_DIR)/100; SOURCE=`ls omi-*.rpm` ; DEST=`echo $$SOURCE | sed 's/r.1././'` ; $(MV) $$SOURCE $$DEST
	cd $(TARGET_DIR)/100; SOURCE=`ls omi-*.deb` ; DEST=`echo $$SOURCE | sed 's/d.1././'` ; $(MV) $$SOURCE $$DEST

	# Gather the SCX bits that we need (due to scx-cimprov versioning; names are fine as is)
	# Note: We take care to only copy the latest version if there are multiple versions
	$(RM) $(TARGET_DIR)/098/scx-*.{rpm,deb} $(TARGET_DIR)/100/scx-*.{rpm,deb}

	cd $(SCX_TARGET_DIR)/098; $(COPY) `ls scx-*.rpm | sort | tail -1` $(TARGET_DIR)/098
	cd $(SCX_TARGET_DIR)/098; $(COPY) `ls scx-*.deb | sort | tail -1` $(TARGET_DIR)/098

	cd $(SCX_TARGET_DIR)/100; $(COPY) `ls scx-*.rpm | sort | tail -1` $(TARGET_DIR)/100
	cd $(SCX_TARGET_DIR)/100; $(COPY) `ls scx-*.deb | sort | tail -1` $(TARGET_DIR)/100

	# Gather the DSC bits that we need
	$(RM) $(TARGET_DIR)/098/omsconfig-*.{rpm,deb} $(TARGET_DIR)/100/omsconfig-*.{rpm,deb}

	cd $(DSC_DIR)/release; $(COPY) `ls omsconfig-*.ssl_098.*.rpm | sort | tail -1` $(TARGET_DIR)/098
	cd $(DSC_DIR)/release; $(COPY) `ls omsconfig-*.ssl_098.*.deb | sort | tail -1` $(TARGET_DIR)/098
	cd $(TARGET_DIR)/098; for f in omsconfig-*.{rpm,deb}; do SOURCE=$$f; DEST=`echo $$SOURCE | sed 's/.ssl_098././'` ; $(MV) $$SOURCE $$DEST; done

	cd $(DSC_DIR)/release; $(COPY) `ls omsconfig-*.ssl_100.*.rpm | sort | tail -1` $(TARGET_DIR)/100
	cd $(DSC_DIR)/release; $(COPY) `ls omsconfig-*.ssl_100.*.deb | sort | tail -1` $(TARGET_DIR)/100
	cd $(TARGET_DIR)/100; for f in omsconfig-*.{rpm,deb}; do SOURCE=$$f; DEST=`echo $$SOURCE | sed 's/.ssl_100././'` ; $(MV) $$SOURCE $$DEST; done

	# Pick up the OSS (Open Source) providers
	$(RMDIR) $(TARGET_DIR)/oss-kits
	$(MKPATH) $(TARGET_DIR)/oss-kits
	cd $(SCX_DIR)/installer/oss-kits; $(COPY) $(OSS_KITS) $(TARGET_DIR)/oss-kits
	cd $(BASE_DIR)/installer/oss-kits; $(COPY) $(OMS_OSS_KITS) $(TARGET_DIR)/oss-kits
	chmod +x $(TARGET_DIR)/oss-kits/*.sh

	# Build the tar file containing both .rpm and .deb packages
	cd $(TARGET_DIR); tar cvf $(OUTPUT_PACKAGE_PREFIX).tar 098/*.{deb,rpm} 100/*.{deb,rpm} oss-kits/*


$(TARGET_DIR)/098/$(OUTPUT_PACKAGE_PREFIX).rpm:
	@echo "========================= Performing Building RPM package (SSL 0.9.8)"
	$(MKPATH) $(INSTALLER_TMPDIR)
	$(MKPATH) $(TARGET_DIR)/098

	sudo $(RMDIR) $(STAGING_DIR)
	python $(PAL_DIR)/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(TARGET_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--RUBY_INT=intermediate/$(BUILD_CONFIGURATION)/098/ruby \
		--RUBY_ARCH=$(RUBY_ARCH) \
		--RUBY_ARCM=$(RUBY_ARCM) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH) \
		--RELEASE=$(OMS_BUILDVERSION_BUILDNR) \
		--VERSION_IDENT="$(OMS_BUILDVERSION_DATE) $(OMS_BUILDVERSION_STATUS)" \
		--DATAFILE_PATH=$(BASE_DIR)/installer/datafiles \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		$(INSTALLER_DATAFILES_RPM)
	mv $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).rpm $(TARGET_DIR)/098

$(TARGET_DIR)/098/$(OUTPUT_PACKAGE_PREFIX).deb:
	@echo "========================= Performing Building DEB package (SSL 0.9.8)"
	$(MKPATH) $(INSTALLER_TMPDIR)
	$(MKPATH) $(TARGET_DIR)/098

	sudo $(RMDIR) $(STAGING_DIR)
	python $(PAL_DIR)/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(TARGET_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--RUBY_INT=intermediate/$(BUILD_CONFIGURATION)/098/ruby \
		--RUBY_ARCH=$(RUBY_ARCH) \
		--RUBY_ARCM=$(RUBY_ARCM) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH) \
		--RELEASE=$(OMS_BUILDVERSION_BUILDNR) \
		--VERSION_IDENT="$(OMS_BUILDVERSION_DATE) $(OMS_BUILDVERSION_STATUS)" \
		$(DPKG_LOCATION) \
		--DATAFILE_PATH=$(BASE_DIR)/installer/datafiles \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		$(INSTALLER_DATAFILES_DPKG)
	mv $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).deb $(TARGET_DIR)/098

$(TARGET_DIR)/100/$(OUTPUT_PACKAGE_PREFIX).rpm:
	@echo "========================= Performing Building RPM package (SSL 1.0)"
	$(MKPATH) $(INSTALLER_TMPDIR)
	$(MKPATH) $(TARGET_DIR)/100

	sudo $(RMDIR) $(STAGING_DIR)
	python $(PAL_DIR)/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(TARGET_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--RUBY_INT=intermediate/$(BUILD_CONFIGURATION)/100/ruby \
		--RUBY_ARCH=$(RUBY_ARCH) \
		--RUBY_ARCM=$(RUBY_ARCM) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH) \
		--RELEASE=$(OMS_BUILDVERSION_BUILDNR) \
		--VERSION_IDENT="$(OMS_BUILDVERSION_DATE) $(OMS_BUILDVERSION_STATUS)" \
		--DATAFILE_PATH=$(BASE_DIR)/installer/datafiles \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		$(INSTALLER_DATAFILES_RPM)
	mv $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).rpm $(TARGET_DIR)/100

$(TARGET_DIR)/100/$(OUTPUT_PACKAGE_PREFIX).deb:
	@echo "========================= Performing Building DEB package (SSL 1.0)"
	$(MKPATH) $(INSTALLER_TMPDIR)
	$(MKPATH) $(TARGET_DIR)/100

	sudo $(RMDIR) $(STAGING_DIR)
	python $(PAL_DIR)/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(TARGET_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--RUBY_INT=intermediate/$(BUILD_CONFIGURATION)/100/ruby \
		--RUBY_ARCH=$(RUBY_ARCH) \
		--RUBY_ARCM=$(RUBY_ARCM) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH) \
		--RELEASE=$(OMS_BUILDVERSION_BUILDNR) \
		--VERSION_IDENT="$(OMS_BUILDVERSION_DATE) $(OMS_BUILDVERSION_STATUS)" \
		$(DPKG_LOCATION) \
		--DATAFILE_PATH=$(BASE_DIR)/installer/datafiles \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		$(INSTALLER_DATAFILES_DPKG)
	mv $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).deb $(TARGET_DIR)/100

else

kit : $(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).$(PACKAGE_SUFFIX)

$(TARGET_DIR)/$(OUTPUT_PACKAGE_PREFIX).$(PACKAGE_SUFFIX) : $(RUBY_DEST_DIR) $(IN_PLUGINS_LIB)

	@echo "========================= Performing Building installation package"
	$(MKPATH) $(INSTALLER_TMPDIR)
	sudo $(RMDIR) $(STAGING_DIR)
	python $(PAL_DIR)/installer/InstallBuilder/installbuilder.py \
		--BASE_DIR=$(BASE_DIR) \
		--TARGET_DIR=$(TARGET_DIR) \
		--INTERMEDIATE_DIR=$(INSTALLER_TMPDIR) \
		--STAGING_DIR=$(STAGING_DIR) \
		--BUILD_TYPE=$(BUILD_TYPE) \
		--BUILD_CONFIGURATION=$(BUILD_CONFIGURATION) \
		--RUBY_INT=intermediate/$(BUILD_CONFIGURATION)/ruby \
		--RUBY_ARCH=$(RUBY_ARCH) \
		--RUBY_ARCM=$(RUBY_ARCM) \
		--PFARCH=$(PF_ARCH) \
		--PFDISTRO=$(PF_DISTRO) \
		--PFMAJOR=$(PF_MAJOR) \
		--PFMINOR=$(PF_MINOR) \
		--VERSION=$(OMS_BUILDVERSION_MAJOR).$(OMS_BUILDVERSION_MINOR).$(OMS_BUILDVERSION_PATCH) \
		--RELEASE=$(OMS_BUILDVERSION_BUILDNR) \
		--VERSION_IDENT="$(OMS_BUILDVERSION_DATE) $(OMS_BUILDVERSION_STATUS)" \
		--DATAFILE_PATH=$(BASE_DIR)/installer/datafiles \
		--OUTPUTFILE=$(OUTPUT_PACKAGE_PREFIX) \
		$(INSTALLER_DATAFILES_CURRENT)

endif

#--------------------------------------------------------------------------------
# Run unit tests
#

tests : test

unittest : $(RUBY_TESTING_DIR)
	@echo "========================= Performing Running MSFT Unit Tests"
	cd $(BASE_DIR); $(RUBY_TESTING_DIR)/bin/rake -f $(BASE_DIR)/test/Rakefile.rb test \
		BASE_DIR=$(BASE_DIR) \
		PLUGINS_TEST_DIR=$(BASE_DIR)/test/code/plugins

systemtest: systemtestrb systemtestsh

systemtestrb : $(RUBY_TESTING_DIR)
	# Any tests that are slow or communicate with a remote server
	@echo "========================= Performing System Test"
	cd $(BASE_DIR); $(RUBY_TESTING_DIR)/bin/rake -f $(BASE_DIR)/test/Rakefile.rb systemtest \
		BASE_DIR=$(BASE_DIR) \
		PLUGINS_TEST_DIR=$(BASE_DIR)/test/code/plugins \
		RUBY_TESTING_DIR=$(RUBY_TESTING_DIR)

systemtestsh :
	@echo "========================= Performing Test omsadmin.sh"
	bash $(BASE_DIR)/test/installer/scripts/test_omsadmin.sh $(BASE_DIR) $(RUBY_TESTING_DIR)


omstest : unittest systemtest

test : omstest
	$(MAKE) -C $(SCX_DIR)/build test
