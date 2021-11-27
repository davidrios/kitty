project(
  'kitty',
  'c', 'cpp',
  version: run_command(
    find_program('python3', 'python'), '-c',
    'from setup import version; print(".".join(str(i) for i in version))'
    ).stdout().strip(),
  meson_version: '>= 0.55.0',
  default_options: [
    'b_ndebug=if-release',
    'buildtype=release',
    'cpp_std=c++11',
    'default_library=shared',
    'warning_level=3',
    'werror=true',
    ])

kitty_version = []
foreach ver: meson.project_version().split('.')
  kitty_version += ver.to_int()
endforeach

cc = meson.get_compiler('c')
cc_ver = []
foreach ver: cc.version().split('.')
  cc_ver += ver.to_int()
endforeach

cpp = meson.get_compiler('cpp')

host_system = host_machine.system()
is_macos = host_system == 'darwin'
is_openbsd = host_system == 'openbsd'

is_msvc = cc.get_id() == 'msvc'
is_gcc = cc.get_id() == 'gcc'

is_debug = get_option('buildtype') == 'debug'
sanitize = get_option('sanitize')
profile = get_option('profile')

cflags = []
cppflags = []
ldflags = []
dependencies = []

if not is_msvc
  cflags += cc.first_supported_argument('-fstack-protector-strong', '-fstack-protector')
else
  cflags += ['/GS', '/RTCs']
endif

if is_gcc and cc_ver[0] <= 5 and cc_ver[1] < 2
  cflags += '-Wno-missing-braces'
endif

df = cc.get_supported_arguments('-g3')

if cc_ver[0] >= 5
  df += cc.get_supported_arguments('-Og')
  cflags += cc.get_supported_arguments('-Wfloat-conversion')
endif

if not (sanitize and is_macos)
  cflags += '-D_FORTIFY_SOURCE=2'
endif

if is_debug or sanitize
  cflags += df
endif

if sanitize
  cflags += cc.get_supported_arguments(
    '-fsanitize=address',
    '-fsanitize=undefined',
    '-fno-omit-frame-pointer')
endif

if is_debug
  cppflags += '-DDEBUG'
else
  cppflags += '-DNDEBUG'
endif

foreach extra_logging: get_option('extra_logging')
  cppflags += '-DDEBUG_' + extra_logging.to_upper().replace('-', '_')
endforeach

copy_file_range = '#define _GNU_SOURCE 1\n#include <unistd.h>\nint main() { copy_file_range(1, NULL, 2, NULL, 0, 0); return 0; }'
if cc.compiles(copy_file_range)
  cflags += '-DHAS_COPY_FILE_RANGE'
endif

if get_option('werror')
  cflags += cc.get_supported_arguments('-pedantic-errors')
endif

native_optimizations = get_option('native_optimizations') and not sanitize and not is_debug
if native_optimizations
  cflags += cc.get_supported_arguments('-march=native')
endif

cflags += cc.get_supported_arguments(
  '-Wextra',
  '-Wno-missing-field-initializers',
  '-Wall',
  '-Wstrict-prototypes',
  '-fwrapv',
  '-fvisibility=hidden')

if not is_debug and not sanitize and not is_openbsd and get_option('link_time_optimization')
  # See https://github.com/google/sanitizers/issues/647
  cflags += cc.get_supported_arguments('-flto')
  ldflags += cc.get_supported_arguments('-flto')
endif

if is_debug
  cflags += '-DKITTY_DEBUG_BUILD'
endif

if profile
  cppflags += '-DWITH_PROFILER'
  cflags += cc.get_supported_arguments('-g3')
  dependencies += cc.find_library('profiler')
endif

cmake = import('cmake')

rsync_dep = cc.find_library('rsync', required: not is_msvc)
if not rsync_dep.found() and is_msvc
  rsync_proj = cmake.subproject('librsync')
  rsync_dep = rsync_proj.dependency('rsync')
endif

rs_sig_args = '''
#include <librsync.h>
int main(void) {
    rs_magic_number magic_number = 0;
    size_t block_len = 0, strong_len = 0;
    rs_sig_args(1024, &magic_number, &block_len, &strong_len);
    return 0;
}'''
if is_msvc or cc.compiles(rs_sig_args, args: cflags, dependencies: rsync_dep)
  cflags += '-DKITTY_HAS_RS_SIG_ARGS'
endif

build_universal_binary = get_option('build_universal_binary') and is_macos
if build_universal_binary
  arches = ['-arch x86_64', '-arch arm64']
  cflags += arches
  ldflags += arches
endif

harfbuzz_dep = dependency('harfbuzz', version: '>= 1.5.0', required: not is_msvc)
if not harfbuzz_dep.found() and is_msvc
  harfbuzz_dep = dependency('harfbuzz', fallback: ['harfbuzz', 'libharfbuzz_dep'])
endif

message('cflags', cflags)
message('cppflags', cppflags)

kitty_sources = [
  'kitty/charsets.c',
  'kitty/child-monitor.c',
  'kitty/child.c',
  'kitty/cleanup.c',
  'kitty/colors.c',
  'kitty/cursor.c',
  'kitty/data-types.c',
  'kitty/disk-cache.c',
  'kitty/fast-file-copy.c',
  'kitty/fonts.c',
  'kitty/gl-wrapper.c',
  'kitty/gl.c',
  'kitty/glfw-wrapper.c',
  'kitty/glfw.c',
  'kitty/glyph-cache.c',
  'kitty/graphics.c',
  'kitty/history.c',
  'kitty/hyperlink.c',
  'kitty/key_encoding.c',
  'kitty/keys.c',
  'kitty/kittens.c',
  'kitty/line-buf.c',
  'kitty/line.c',
  'kitty/logging.c',
  'kitty/loop-utils.c',
  'kitty/monotonic.c',
  'kitty/mouse.c',
  'kitty/parser.c',
  'kitty/png-reader.c',
  'kitty/ringbuf.c',
  'kitty/screen.c',
  'kitty/shaders.c',
  'kitty/state.c',
  'kitty/unicode-data.c',
  'kitty/utmp.c',
  'kitty/wcswidth.c',
]

if is_macos
  kitty_sources += [
    'kitty/core_text.m',
    'kitty/cocoa_window.m',
    'kitty/macos_process_info.c'
  ]
elif is_msvc
  # TODO
else
  kitty_sources += [
    'kitty/fontconfig.c',
    'kitty/freetype.c',
    'kitty/desktop.c',
    'kitty/freetype_render_ui_text.c',
  ]
endif

kitty_headers = [
  'kitty/banned.h',
  'kitty/charsets.h',
  'kitty/cleanup.h',
  'kitty/colors.h',
  'kitty/control-codes.h',
  'kitty/cross-platform-random.h',
  'kitty/data-types.h',
  'kitty/disk-cache.h',
  'kitty/emoji.h',
  'kitty/fast-file-copy.h',
  'kitty/fonts.h',
  'kitty/freetype_render_ui_text.h',
  'kitty/gl-wrapper.h',
  'kitty/gl.h',
  'kitty/glfw-wrapper.h',
  'kitty/glyph-cache.h',
  'kitty/graphics.h',
  'kitty/hyperlink.h',
  'kitty/iqsort.h',
  'kitty/keys.h',
  'kitty/kitty-uthash.h',
  'kitty/lineops.h',
  'kitty/loop-utils.h',
  'kitty/modes.h',
  'kitty/monotonic.h',
  'kitty/parse-graphics-command.h',
  'kitty/png-reader.h',
  'kitty/rewrap.h',
  'kitty/ringbuf.h',
  'kitty/safe-wrappers.h',
  'kitty/screen.h',
  'kitty/state.h',
  'kitty/threading.h',
  'kitty/unicode-data.h',
  'kitty/uthash.h',
  'kitty/wcswidth.h',
  'kitty/wcwidth-std.h',
]

python = import('python')
py_inst = python.find_installation('python3')

parser_dump = custom_target(
  'parser_dump.c',
  build_by_default: true,
  output: 'parser_dump.c',
  input: 'kitty/parser.c',
  capture: true,
  command: [
    py_inst,
    '-c',
    'import sys; sys.stdout.write(open(sys.argv[1]).read().replace("DUMP_COMMANDS", "ALWAYS_TRUE"))',
    '@INPUT@'
  ]
)

kitty_sources += parser_dump

kitty_flags = cflags
kitty_cppflags = cppflags

kitty_cppflags += [
  '-DPRIMARY_VERSION=' + (kitty_version[0] + 4000).to_string(),
  '-DSECONDARY_VERSION=' + kitty_version[1].to_string(),
  '-DXT_VERSION="@0@"'.format(meson.project_version()),
  '-DALWAYS_TRUE'
]

kitty_deps = dependencies
kitty_deps += py_inst.dependency(embed: true)

if is_macos
  # TODO
else
  kitty_deps += dependency('fontconfig')
endif

kitty_deps += [
  harfbuzz_dep,
  dependency('libpng'),
  dependency('lcms2'),
  dependency('gl'),
  dependency('zlib'),
  dependency('threads'),
  cc.find_library('m', required: false)
]

if not is_openbsd
  kitty_deps += [
    cc.find_library('rt'),
    cc.find_library('dl')
  ]
endif

library(
  'fast_data_types',
  kitty_headers + kitty_sources,
  c_args: kitty_flags + kitty_cppflags,
  dependencies: kitty_deps,
  name_prefix: '',
  include_directories: include_directories('kitty'))
