# SPDX-License-Identifier: GPL-2.0-or-later
# Copyright (c) Iputils Project, 2017-2024

project('iputils', 'c',
	default_options : [
		'c_std=c99',
		'warning_level=3',
		'localstatedir=var',
	],
	meson_version : '>=0.44', # >=0.58 is required for 'meson dist'
	version : '20250605') # keep in sync with: git describe | awk -F- '{print $1}'

cc = meson.get_compiler('c')

add_project_arguments(
	'-include', 'config.h',
	'-include', 'git-version.h',
	language : 'c'
)

conf = configuration_data()
conf.set_quoted('PACKAGE_NAME', meson.project_name())

build_arping = get_option('BUILD_ARPING')
build_clockdiff = get_option('BUILD_CLOCKDIFF')
build_ping = get_option('BUILD_PING')
build_tracepath = get_option('BUILD_TRACEPATH')

build_mans = get_option('BUILD_MANS')
build_html_mans = get_option('BUILD_HTML_MANS')
run_tests = not get_option('SKIP_TESTS')

prefix = get_option('prefix')
bindir = join_paths(prefix, get_option('bindir'))
localedir = join_paths(prefix, get_option('localedir'))
sbindir = join_paths(prefix, get_option('sbindir'))

systemd = dependency('systemd', required : false)
install_systemd_units = systemd.found() or get_option('INSTALL_SYSTEMD_UNITS')

systemdunitdir = get_option('systemdunitdir')
if systemdunitdir == '' and install_systemd_units
	systemdunitdir = systemd.get_pkgconfig_variable('systemdsystemunitdir')
endif

setcap_arping = false
setcap_clockdiff = false
setcap_ping = false
if (not get_option('NO_SETCAP_OR_SUID'))
  setcap_arping = get_option('SETCAP_OR_SUID_ARPING')
  setcap_clockdiff = get_option('SETCAP_OR_SUID_CLOCKDIFF')
  setcap_ping = get_option('SETCAP_OR_SUID_PING')
endif

# Check functions.
foreach f : '''
	__fpending
	getrandom
	nanosleep
'''.split()
	if cc.has_function(f, args : '-D_GNU_SOURCE')
		conf.set('HAVE_' + f.to_upper(), 1,
			description : 'Define to 1 if you have the ' + f + ' function')
	endif
endforeach

if cc.compiles('''
	#include <sys/types.h>
	#include <netinet/in.h>
	#include <netinet/icmp6.h>
	int main(void) {
		struct icmp6_nodeinfo nodeinfo;
		return 0;
	}
''', name : 'struct icmp6_nodeinfo')
	conf.set('HAVE_STRUCT_ICMP6_NODEINFO', 1,
		description : 'Defined if struct icmp6_nodeinfo exists.')
endif

if cc.compiles('''
	#include <sys/types.h>
	#include <sys/time.h>
	#include <time.h>
	int main(void) {
		struct tm time;
		return 0;
	}
''', name : 'struct tm time;')
	conf.set('TIME_WITH_SYS_TIME', 1,
		description : 'Defined if struct tm exists.')
endif

m_dep = cc.find_library('m')
resolv_dep = cc.find_library('resolv')
if cc.has_function('clock_gettime')
	rt_dep = cc.find_library('disabler-appears-to-disable-executable-build', required : false)
else
	rt_dep = cc.find_library('rt')
endif

cap = get_option('USE_CAP')
if cap == true
	cap_dep = cc.find_library('cap')
	conf.set('HAVE_LIBCAP', 1,
		description : 'Defined if libcap exists.')
else
	cap_dep = dependency('disabler-appears-to-disable-executable-build', required : false)
endif

opt = get_option('ARPING_DEFAULT_DEVICE')
if opt != ''
	conf.set_quoted('DEFAULT_DEVICE', opt, description : 'arping default device.')
endif

# Some systems needs intl for gettext
intl_dep = []
gettext = get_option('USE_GETTEXT')
if gettext == true
	add_project_arguments(['-DGETTEXT_PACKAGE="iputils"'], language: 'c')

	conf.set('ENABLE_NLS', 1, description : 'If set enable I18N.')
	conf.set_quoted('LOCALEDIR', localedir)
	subdir ('po')
	intl_dep += cc.find_library('intl', required : false)
endif

idn = get_option('USE_IDN')
if idn == true
	idn_dep = dependency('libidn2', required : false)
	if idn_dep.found()
		add_project_arguments('-DUSE_IDN', language : 'c')
		conf.set('USE_IDN', 1,
			description : 'If set use Internationalized Domain Name library.')
	endif
else
	idn_dep = dependency('disabler-appears-to-disable-executable-build', required : false)
endif

if host_machine.endian() == 'big'
	conf.set('WORDS_BIGENDIAN', '1')
endif

foreach h : [
	'error.h',
]
	if cc.has_header(h)
		conf.set('HAVE_' + h.to_upper().underscorify(), 1,
			description : 'Define if ' + h + ' can be included.')
	endif
endforeach

git_version_h = vcs_tag(
	input : 'git-version.h.meson',
	output : 'git-version.h',
	fallback : meson.project_version()
)

config_h = configure_file(
	output : 'config.h',
	configuration : conf)

setcap = find_program('setcap', '/usr/sbin/setcap', '/sbin/setcap', required : false)
if cap_dep.found() and setcap.found()
	perm_type = 'caps'
	setcap_path = setcap.path()
else
	perm_type = 'setuid'
	setcap_path = '/dev/null'
endif

############################################################
common_sources = files(
	'iputils_common.h', 'iputils_common.c',
	'md5.h', 'md5.c'
)
libcommon = static_library(
	'common',
	[common_sources, git_version_h],
	install : false)

if build_ping == true
	subdir ('ping')
endif

if build_tracepath == true
	tracepath = executable('tracepath', ['tracepath.c', git_version_h],
		dependencies : [idn_dep, intl_dep],
		link_with : [libcommon],
		install: true)
endif

if build_clockdiff == true
	clockdiff = executable('clockdiff', ['clockdiff.c', git_version_h],
		dependencies : [cap_dep, intl_dep],
		link_with : [libcommon],
		install: true)
	if (setcap_clockdiff)
		meson.add_install_script('build-aux/setcap-setuid.sh',
			bindir,
			'clockdiff',
			perm_type,
			setcap_path
		)
	endif
endif

if build_arping == true
	arping = executable('arping', ['arping.c', git_version_h],
		dependencies : [rt_dep, cap_dep, idn_dep, intl_dep],
		link_with : [libcommon],
		install: true)
	if (setcap_arping)
		meson.add_install_script('build-aux/setcap-setuid.sh',
			bindir,
			'arping',
			perm_type,
			setcap_path
		)
	endif
endif

if build_mans == true or build_html_mans == true
	subdir ('doc')

	if meson.version().version_compare('>=0.48.0')
		meson.add_dist_script('build-aux/meson-build-dist-man.sh')
	endif
endif

if run_tests
	subdir('test')
endif

output = '\n\nAPPLICATIONS (build)\n'
output += 'man: ' + build_mans.to_string() + '\n'
output += 'HTML man: ' + build_html_mans.to_string() + '\n'
output += 'arping: ' + build_arping.to_string()
output += ' (capability or suid: ' + setcap_arping.to_string() + ')\n'
output += 'clockdiff: ' + build_clockdiff.to_string()
output += ' (capability or suid: ' + setcap_clockdiff.to_string() + ')\n'
output += 'ping: ' + build_ping.to_string()
output += ' (capability or suid: ' + setcap_ping.to_string() + ')\n'
output += 'tracepath: ' + build_tracepath.to_string() + '\n'

output += '\nCONFIGURATION\n'
output += 'Capability (with libcap): ' + cap.to_string() + '\n'
output += 'IDN (with libidn2): ' + idn.to_string() + '\n'
output += 'I18N (with gettext): ' + gettext.to_string() + '\n'
output += 'systemd: ' + install_systemd_units.to_string() + '\n'
output += 'run tests: ' + run_tests.to_string() + '\n'

output += '\nSYSTEM PATHS\n'
output += 'prefix: ' + prefix + '\n'
output += 'bindir: ' + bindir + '\n'
output += 'localedir: ' + localedir + '\n'
output += 'sbindir: ' + sbindir + '\n'
output += 'systemdunitdir: ' + systemdunitdir + '\n'

message(output)
