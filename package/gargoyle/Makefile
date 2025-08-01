include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk


PKG_NAME:=gargoyle
PKG_VERSION:=$(GARGOYLE_VERSION)
ifeq ($(GARGOYLE_VERSION),)
     PKG_VERSION:=1.0.0
endif
PKG_RELEASE:=7

ATH_DEPS=
LED_DEPS=
BRCM_DEPS=
ifeq ($(BOARD),atheros)
     ATH_DEPS=+wpa-supplicant +hostapd
     LED_DEPS=+gpioctl
endif

ifeq ($(BOARD),brcm-2.4)
     BRCM_DEPS=+wl +bin2trx
endif

PKG_BUILD_DIR := $(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk

define Package/gargoyle
	SUBMENU:=Gargoyle Web Interface
	SECTION:=admin
	CATEGORY:=Administration
	TITLE:=Gargoyle Web Interface
	DEPENDS:=+uci +libuci +libjson-c +haserl-i18n +uhttpd +libustream-mbedtls +bwmon-gargoyle +webmon-gargoyle +libericstools +gpkg +gargoyle-ip-query  $(ATH_DEPS) $(BRCM_DEPS) $(LED_DEPS)
	MAINTAINER:=Eric Bishop <eric@gargoyle-router.com>
endef

define Package/gargoyle/description
	A user-friendly web interface for OpenWrt
endef

define Package/gargoyle/config
	select PACKAGE_gargoyle-firewall-util
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Compile
	$(MAKE) -C $(PKG_BUILD_DIR) \
		$(TARGET_CONFIGURE_OPTS) \
		STAGING_DIR="$(STAGING_DIR)" \
		CFLAGS="$(TARGET_CFLAGS) -I $(STAGING_DIR)/usr/include" \
		LDFLAGS="$(TARGET_LDFLAGS) -L $(STAGING_DIR)/usr/lib" 
endef

# escape commas, backslashes, squotes, and ampersands for sed
define sed_escape
$(subst &,\&,$(subst $(comma),\$(comma),$(subst ','\'',$(subst \,\\,$(1)))))
endef
#'

define Package/gargoyle/install
	$(INSTALL_DIR) $(1)
	$(CP) ./files/* $(1)/
	mkdir -p $(1)/www/data/
	mkdir -p $(1)/www/hooks/
	mkdir -p $(1)/www/hooks/login/

	( \
		if [ -n "$(OFFICIAL_VERSION)" ] ; then \
			echo "$(OFFICIAL_VERSION)" > $(1)/www/data/gargoyle_version.txt ; \
		else \
			echo "$(PKG_VERSION)-custom (Built $(shell date -u))" > $(1)/www/data/gargoyle_version.txt ; \
		fi \
	)


	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/gargoyle_header_footer $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/gargoyle_session_validator $(1)/usr/bin
	$(SED) 's,%CURYEAR,$(shell date -u +%Y),g' $(1)/www/about.sh
	$(SED) 's,%GV,$(call sed_escape,$(GARGOYLE_DISTRIB_RELEASE)),g' \
		-e 's,%GR,$(call sed_escape,$(GARGOYLE_DISTRIB_REVISION)),g' \
		-e 's,%GT,$(call sed_escape,$(GARGOYLE_DISTRIB_TARGET)),g' \
		-e 's,%GP,$(call sed_escape,$(GARGOYLE_DISTRIB_PROFILE)),g' \
		-e 's,%GD,$(call sed_escape,$(GARGOYLE_DISTRIB_DESCRIPTION)),g' \
		$(1)/etc/gargoyle_release
endef


define Package/gargoyle/postinst

	#if this is brcm-2.4, remove client bridge hotplug script, this isn't necessary for brcm-2.4
	if [ -e $${IPKG_INSTROOT}/usr/sbin/nas ] ; then
		rm $${IPKG_INSTROOT}/etc/hotplug.d/net/20-client_bridge 
	fi
	
	
	#if this is atheros device, set uci options that set fonera power led to on
	#this may cause further weirdness with other atheros devices, but quite frankly, I don't give a shit
	#fons are by far the most popular devices with atheros cpus
	if [ -n "$(LED_DEPS)" ] ; then
		echo "" >> $${IPKG_INSTROOT}/etc/config/system
		echo "config led power" >> $${IPKG_INSTROOT}/etc/config/system
		echo "	option sysfs gpio7" >> $${IPKG_INSTROOT}/etc/config/system
		echo "	option default 1" >> $${IPKG_INSTROOT}/etc/config/system
		echo "" >> $${IPKG_INSTROOT}/etc/config/system
	fi

	#if we're installing as a package on an active openwrt system, initialize by doing the following:
	if [ -z "$${IPKG_INSTROOT}" ] ; then
		#if qos enabled, initialize status chart
		qos_enabled=$$(ls $${IPKG_INSTROOT}/etc/rc.d/*qos_gargoyle* 2>/dev/null)
		if [ -n "$$qos_enabled" ] ; then
			uci set gargoyle.status.qos=300 2>/dev/null
		fi

		#restart web server and run ntp client update
		/etc/init.d/uhttpd restart 2>/dev/null
		/etc/init.d/sysntpd restart 2>/dev/null
	fi
endef

$(eval $(call BuildPackage,gargoyle))
