project('mpdecimal', 'c', 'cpp',
        version : '2.5.0',
        license : 'BSD-2-Clause',
        default_options: [
          'c_std=c99',
          'cpp_std=c++11',
        ])

library_version = meson.project_version()
v = library_version.split('.')
compat_version = '.'.join([v[0], v[1]])

cc = meson.get_compiler('c')
m_dep = cc.find_library('m', required : false)

conf = configuration_data()

have_uint128_t = cc.sizeof('__uint128_t') > 0

# x64 with gcc asm:
gcc_asm_for_x64 = cc.compiles('''
  int main()
  {
    __asm__ __volatile__ ("movq %rcx, %rax");
  }''',
  name : 'x64 gcc inline assembler')

# x87 with gcc asm:
gcc_asm_for_x87 = cc.compiles('''
  int main()
  {
    unsigned short cw;
    __asm__ __volatile__ ("fnstcw %0" : "=m" (cw));
    __asm__ __volatile__ ("fldcw %0" : : "m" (cw));
  }''',
  name : 'x87 gcc inline assembler')

# _FORTIFY_SOURCE wrappers for memmove and bcopy are incorrect:
# http://sourceware.org/ml/libc-alpha/2010-12/msg00009.html
if cc.run('''
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  void foo(void *p, void *q) { memmove(p, q, 19); }
  int main() {
    char a[32] = "123456789000000000";
    foo(&a[9], a);
    if (strcmp(a, "123456789123456789000000000") != 0)
      return 1;
    foo(a, &a[9]);
    if (strcmp(a, "123456789000000000") != 0)
      return 1;
    return 0;
  }''',
  args : ['-O2', '-D_FORTIFY_SOURCE=2'],
  name : 'glibc _FORTIFY_SOURCE/memmove').returncode() != 0

  warning('''
***************************** WARNING *********************************

Detected glibc _FORTIFY_SOURCE/memmove bug. See:

    http://sourceware.org/ml/libc-alpha/2010-12/msg00009.html

Enabling -U_FORTIFY_SOURCE workaround. If -D_FORTIFY_SOURCE is also
present in the command line, make sure that the order of the two
options is:

   ... -D_FORTIFY_SOURCE=2 ... -U_FORTIFY_SOURCE ...

A better solution is to upgrade glibc or to report the bug to your
OS vendor.

***************************** WARNING *********************************
''')
  add_project_arguments('-U_FORTIFY_SOURCE', language: 'c')
endif

# Some versions of gcc miscompile inline asm:
# http://gcc.gnu.org/bugzilla/show_bug.cgi?id=46491
# http://gcc.gnu.org/ml/gcc/2010-11/msg00366.html
if cc.get_id() == 'gcc' and cc.run('''
  __attribute__((noinline)) int
  foo(int *p) {
    int r;
    asm ( "movl \$6, (%1)\n\t"
          "xorl %0, %0\n\t"
          : "=r" (r) : "r" (p) : "memory"
    );
    return r;
  }
  int main() {
    int p = 8;
    if ((foo(&p) ? : p) != 6)
      return 1;
    return 0;
  }''',
  args : ['-O2'],
  name : 'gcc ipa-pure-const').returncode() != 0

  add_project_arguments('-fno-ipa-pure-const', language: 'c')
endif

# Values for the MPD_HEADER_CONFIG variable:
mpd_header_config_64 = '''
/* ABI: 64-bit */
#ifdef CONFIG_32
  #error "cannot use CONFIG_32 with 64-bit header."
#endif

#ifndef CONFIG_64
  #define CONFIG_64
#endif
'''
mpd_header_config_32 = '''
/* ABI: 32-bit */
#ifdef CONFIG_64
  #error "cannot use CONFIG_64 with 32-bit header."
#endif

#ifndef CONFIG_32
  #define CONFIG_32
#endif
'''
mpd_header_config_32_legacy = '''
/* ABI: 32-bit */
#ifdef CONFIG_64
  #error "cannot use CONFIG_64 with 32-bit header."
#endif

#ifndef CONFIG_32
  #define CONFIG_32
#endif

/* libmpdec was compiled without support for int64_t */
#ifndef LEGACY_COMPILER
  #define LEGACY_COMPILER
#endif"
'''
mpd_header_config_universal = '''
/* Mac OS X: support for building universal binaries */
#if defined(CONFIG_64) || defined(CONFIG_32)
  #error "cannot use CONFIG_64 or CONFIG_32 with universal header."
#endif

#if defined(__ppc__)
  #define CONFIG_32
  #define ANSI
#elif defined(__ppc64__)
  #define CONFIG_64
  #define ANSI
#elif defined(__i386__)
  #define CONFIG_32
  #define ANSI
#elif defined(__x86_64__)
  #define CONFIG_64
  #define ASM
#else
  #error "unknown architecture for universal build."
#endif
'''
mpd_header_config_fullcov = '''
/* DO NOT USE IN PRODUCTION! */
#if defined(CONFIG_64) || !defined(CONFIG_32)
  #error "CONFIG_32 must be defined for full coverage."
#endif
'''

machine = get_option('machine')
if machine == 'auto'
  if host_machine.system() == 'windows'
    if host_machine.cpu_family() == 'x86'
      machine = 'ppro'
    elif host_machine.cpu_family() == 'x86_64'
      machine = 'x64'
    else
      error('machine probing on Windows only supports x86/x64')
    endif
  else
    if cc.sizeof('size_t') == 8
      if gcc_asm_for_x64
        machine = 'x64'
      elif have_uint128_t
        machine = 'uint128'
      else
        machine = 'ansi64'
      endif
    else
      if cc.get_id() == 'ccomp'
        machine = 'ansi-legacy'
      else
        machine = 'ansi32'
      endif
      if gcc_asm_for_x87
        if cc.get_id() == 'gcc' or cc.get_id() == 'clang'
          if target_machine.system() != 'darwin'
            machine = 'ppro'
          endif
        endif
      endif
    endif
  endif
endif

if machine == 'x64'
  add_project_arguments('-DCONFIG_64', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_64)
  if host_machine.system() == 'windows'
    add_project_arguments('/DMASM', language: 'c')
  else
    add_project_arguments('-DASM', language: 'c')
  endif

elif machine == 'uint128'
  add_project_arguments('-DCONFIG_64', '-DANSI', '-DHAVE_UINT128_T', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_64)

elif machine == 'ansi64'
  add_project_arguments('-DCONFIG_64', '-DANSI', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_64)

elif machine == 'full-coverage'
  add_project_arguments('-DCONFIG_32', '-DANSI', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_fullcov)

elif machine == 'ppro'
  add_project_arguments('-DCONFIG_32', '-DPPRO', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_32)
  if host_machine.system() == 'windows'
    add_project_arguments('/DMASM', language: 'c')
  endif

elif machine == 'ansi32'
  add_project_arguments('-DCONFIG_32', '-DANSI', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_32)

elif machine == 'ansi-legacy'
  add_project_arguments('-DCONFIG_32', '-DANSI', '-DLEGACY_COMPILER', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_32_legacy)

elif machine == 'universal'
  add_project_arguments('-DUNIVERSAL', language : 'c')
  conf.set('MPD_HEADER_CONFIG', mpd_header_config_universal)
endif

build_dll = []
use_dll = []
if host_machine.system() == 'windows'
  add_project_arguments('/wd4200', '/wd4204', '/wd4221', '/D_CRT_SECURE_NO_WARNINGS', '/nologo', language: 'c')
  if get_option('default_library') != 'static'
    build_dll = ['/DBUILD_DLL']
    use_dll = ['/DUSE_DLL']
  endif
  vcver = machine == 'x64' ? '64vc' : '32vc'
  mpdecimal_h = configure_file(
    input: 'libmpdec/mpdecimal@0@.h'.format(vcver),
    output : 'mpdecimal.h',
    copy: true)
  install_headers('libmpdec/vccompat.h')
else
  mpdecimal_h = configure_file(
    input : 'libmpdec/mpdecimal.h.in',
    output : 'mpdecimal.h',
    configuration : conf)
endif

libmpdec_sources = files(
  'libmpdec/basearith.c',
  'libmpdec/context.c',
  'libmpdec/constants.c',
  'libmpdec/convolute.c',
  'libmpdec/crt.c',
  'libmpdec/mpdecimal.c',
  'libmpdec/mpsignal.c',
  'libmpdec/difradix2.c',
  'libmpdec/fnt.c',
  'libmpdec/fourstep.c',
  'libmpdec/io.c',
  'libmpdec/mpalloc.c',
  'libmpdec/numbertheory.c',
  'libmpdec/sixstep.c',
  'libmpdec/transpose.c',
)

if host_machine.system() == 'windows' and machine == 'x64'
  ml = find_program('ml64')
  libmpdec_sources += custom_target(
    'vcdiv64.asm.obj',
    input: 'libmpdec/vcdiv64.asm',
    output: 'vcdiv64.asm.obj',
    command: [ml, '/Cx', '/Fo', '@OUTPUT@', '/c', '@INPUT@'])
endif

libmpdec = library(
  'mpdec',
  libmpdec_sources,
  c_args: build_dll,
  dependencies : m_dep,
  version : library_version,
  darwin_versions : [compat_version, library_version])

mpdec_dep = declare_dependency(
  link_with : libmpdec,
  compile_args: use_dll,
  # mpdecimal.h is generated in the project root directory
  include_directories : include_directories('.', 'libmpdec'))

libmpdecpp_sources = files(
  'libmpdec++/decimal.cc',
)

libmpdecpp = library(
  'mpdec++',
  libmpdecpp_sources, mpdecimal_h,
  dependencies : [m_dep, mpdec_dep],
  version : library_version,
  darwin_versions : [compat_version, library_version])

install_headers(mpdecimal_h, 'libmpdec++/decimal.hh')

mpdecpp_dep = declare_dependency(
  link_with : libmpdecpp,
  include_directories : include_directories('libmpdec++'),
  dependencies : mpdec_dep)
